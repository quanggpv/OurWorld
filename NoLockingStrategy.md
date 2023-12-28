<b>Xử lý các truy cập đồng thời nhưng không dùng lock transaction của database.</b>

Bài toán:

Dev: Sếp ơi, phần mua hàng này sẽ có trường hợp nhiều người dùng cùng mua một lúc, em cần lock resource này lại để đảm bảo dữ liệu đúng đắn & đúng với thực tiễn,
em sẽ dùng lock trong database để xử lý việc này, sếp nghĩ sao ạ?
 
Lead: Cũng là một cách, nhưng hiện tại phần này liên quan đến đặt hàng, anh không muốn có xác suất tạo ra `deadlock`, nếu xảy ra sẽ ảnh hưởng rất nhiều đến những services khác (rủi ro cao),
em thử nghĩ cách khác đi. 

---------------------------------------------------------------------------------------------------------------------------------------------------
# Định nghĩa các tables:

Inventory (id, product_variant_id, quantity, ...)

Order (id, ...)

order_detail (id, order_id, product_variant_id, ...)

Products (id, name, ...)

ProductVariants (id, name, product_id,...)

`quan hệ các bảng` <br>

Order - order_detail: 1-n <br>

Products - ProductVariants: 1-n <br>

ProductVariants - Inventory: 1-1 <br>

# Một chút về mức độ cô lập (isolation level)
Mặc định mức độ cô lập (isolation level) của  mariadb/mysql là REPEATABLE READ
- REPEATABLE READ: Bạn chỉ thể đọc data từ query trước đó ngày cả khi bạn query nhiều lần cùng một câu (đó là ý của từ "REPEAT") 
- READ COMMITTED: Bạn sẽ đọc dược những data đã commit

Mình sẽ diễn giải thêm ở đây
```
BEGIN TRANSACTION: 
$query = "SELECT quantity FROM INVENTORY WHERE ID = 1"
//Lúc này quantity = 10;

sleep(3) 

//Luồng khác update quantity bằng 100 (đã thực hiện commit)
$query = "SELECT * FROM INVENTORY WHERE ID = 1" 

// dùng `REPEATABLE READ`, nên query tiếp theo khi đọc quantity sẽ là 10
// dùng `READ COMMITTED`, nên query tiếp theo khi đọc quantity sẽ là 100

COMMIT
```
# Bắt đầu xử lý



1. Tìm thông tin tồn kho của một sản phẩm A (trước khi đặt hàng) <br>
2. Số lượng của sản phẩm này chỉ còn 01 sản phẩm nhưng đang có 02 người mua cùng lúc<br>

- User U1 mua tại `thời điểm T1`, tồn kho của sản phẩm A có id là `$productVariantId=1` và có `số lượng là 999` <br>
- User U2 mua tại `thời điểm T1`, tồn kho của sản phẩm A có id là `$productVariantId=1` và có `số lượng là 999` <br>
(`$productVariantId` và `Số lượng` sẽ là thành phần mình dùng để làm checkpoint) <br>


Logic code xử lý đồng thời mà không lock row của mình.
```
//id của sản phẩm (sku)
$productVariantId = 1;

$quantityFromUser = 1;

// Ở đây mình đổi isolation level về `READ COMMITTED`
// (Nên set ở transaction cấp cao nhất)
DB::unprepared('SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED');
try {
DB::beginTransaction();
  //cho retry 5 lần
  $retryLimit = 5;
  $retryCount = 0;
  $affectedRow = 0;

  while ($retryCount < $retryLimit) {
      //câu select query này bị ảnh hưởng bởi isolation level của db

      // lấy thông tin tồn kho mới nhất
      $inventory = Inventory::query()->where([
          'product_variant_id' => productVariantId
      ])->first();

      // Số lượng mong muốn cập nhật
      $quantityExpected = $inventory->quantity - $quantityfromUser;

      // hết hàng trong tồn kho
      if ($quantityExpected < 0) {
          // xử lý exception
      }

      // Dựa vào hai điều kiện mình đánh checkpoint trước đó để cập nhật
      // Vì có thể có một luồng khác cập nhật dữ liệu (U1 mua trước U2 hoặc ngược lại), 
      // dẫn tới giá trị lúc này đang tính toán sẽ là rác, mang đi cập nhật sẽ không còn đúng (trường hợp 'lost-update') 
      // Nếu:
      // $affectedRow === 0; // Tức bị người khác (U1 hoặc U2) thay đổi giá trị quantity, nên cần chạy lại (retry) để lấy thông tin tồn kho mới nhất
      // $affectedRow > 0; // cập nhật thành công và không bị thay đổi dữ liệu bởi luồng khác
      $affectedRow = Inventory::where([
          'product_variant_id' => productVariantId,
          'quantity' => $inventoryByProductVariantId->quantity,
      ])->update([
          'quantity' => $quantityExpected,
          'updated_at' => Carbon::now()->format('Y-m-d H:i:s'),
          'updated_by' => isset($user) && isset($user->email) ? $user->email : null,
      ]);

      //cập nhật thành công và không bị thay đổi dữ liệu bởi luồng khác
      if ($affectedRow > 0) {
          break;
      }

     
      $retryCount++;
  }

  // Khi vượt quá lần retry thì cân nhắc trả lỗi cho người dùng, để người dùng thực hiện chức năng khác, hoặc lặp lại việc đặt hàng
  if ($affectedRow === 0) {
      throw new \Exception('Thao tác thất bại do tài nguyên đang được sử dụng bởi người khác, hãy chạy lại chức năng');
  }
  DB::commit();
} catch (Exception $e) {
  DB::rollBack();
}
//Set về ISOLATION LEVEL ban đầu là REPEATABLE READ
DB::unprepared('SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ');



```


Done.

Đó là cách mình đã làm để xử lý đồng thời mà không dùng locking của database, nếu bạn có cách nào khác thì hãy chia sẽ nhé, thank u.


