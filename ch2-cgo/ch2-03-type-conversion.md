# 2.3 Chuyển đổi kiểu

Ban đầu, CGO được tạo ra để thuận lợi cho việc sử dụng các hàm trong C (các hàm hiện thực khai báo Golang trong C) để sử dụng lại các tài nguyên của C (vì ngôn ngữ C cũng liên quan đến các hàm callback, dĩ nhiên nó liên quan đến việc gọi các hàm trong Go từ các hàm của C (các hàm thực hiện khai báo ngôn ngữ C trong Go)). Ngày nay, CGO đã phát triển thành cầu nối giao tiếp hai chiều giữa C và Go. Để tận dụng tính năng của CGO, việc hiểu các quy tắc chuyển đổi kiểu dữ liệu giữa hai loại ngôn ngữ là điều quan trọng. Đây là vấn đề sẽ được thảo luận trong phần này.

## 2.3.1 Các kiểu dữ liệu số học

Khi ta sử dụng các ký hiệu của C trong Golang, thường nó sẽ truy cập thông qua package "C" ảo, chẳng hạn như kiểu `int` tương ứng với `C. int`. Một số kiểu trong C bao gồm nhiều từ khóa, nhưng khi truy cập chúng thông qua package "C" ảo, phần tên không thể có ký tự khoảng trắng, ví dụ `unsigned int` không thể truy cập trực tiếp `C.unsigned int`. Do đó, CGO cung cấp quy tắc chuyển đổi tương ứng cho các kiểu trong C cơ bản, ví dụ như `C.uint` tương ứng trong C là `unsigned int`.

Kiểu dữ liệu số học và kiểu dữ liệu trong C của Golang về cơ bản là tương tự nhau. Bảng 2-1 thể hiện sự tương tự này.

| Kiểu trong C           | Kiểu trong CGO | Kiểu trong Go |
| ---------------------- | -------------- | ------------- |
| char                   | C.char         | byte          |
| singed char            | C.schar        | int8          |
| unsigned char          | C.uchar        | uint8         |
| short                  | C.short        | int16         |
| unsigned short         | C.ushort       | uint16        |
| int                    | C.int          | int32         |
| unsigned int           | C.uint         | uint32        |
| long                   | C.long         | int32         |
| unsigned long          | C.ulong        | uint32        |
| long long int          | C.longlong     | int64         |
| unsigned long long int | C.ulonglong    | uint64        |
| float                  | C.float        | float32       |
| double                 | C.double       | float64       |
| size_t                 | C.size_t       | uint          |

_Bảng 2-1 So sánh kiểu trong các ngôn ngữ Go và C_

Cần lưu ý rằng mặc dù kích thước của những kiểu không được chỉ rõ kích thước trong C như `int`, `short` v.v., kích thước của chúng đều được xác định trong CGO. Trong CGO, kiểu `int` và `uint` của C đều có kích thước 4 byte, kiểu `size_t` có thể được coi là kiểu số nguyên không dấu `uint` của ngôn ngữ Go .

Mặc dù kiểu `int` và `uint` của C đều có kích thước cố định, nhưng với GO thì `int` và `uint` có thể là 4 byte hoặc 8 byte. Nếu cần sử dụng đúng kiểu `int` của C trong Go, bạn có thể sử dụng kiểu `GoInt` được xác định trong file header `_cgo_export.h` được tạo ra bởi công cụ CGO. Trong file header này, mỗi kiểu giá trị Go cơ bản sẽ xác định kiểu tương ứng trong C có tiền tố "Go". Ví dụ sau trong hệ thống 64-bit, có file header `_cgo_export.h` được CGO định nghĩa các kiểu giá trị, nơi mà `GoInt` và `GoUint` lần lượt là `GoInt64` và `GoUint64`:

```go
typedef signed char GoInt8;
typedef unsigned char GoUint8;
typedef short GoInt16;
typedef unsigned short GoUint16;
typedef int GoInt32;
typedef unsigned int GoUint32;
typedef long long GoInt64;
typedef unsigned long long GoUint64;
typedef GoInt64 GoInt;
typedef GoUint64 GoUint;
typedef float GoFloat32;
typedef double GoFloat64;
```

Bên cạnh `GoInt` và `GoUint`, chúng tôi không khuyên bạn nên sử dụng trực tiếp `GoInt32`, `GoInt64` và các kiểu khác. Cách tiếp cận tốt hơn là khai báo file header <stdint.h> thông qua tiêu chuẩn C99 của C. Để cải thiện tính linh hoạt của C, không chỉ mỗi kiểu số học được xác định kích thước rõ ràng trong file mà chúng còn sử dụng các tên phù hợp với tên kiểu tương ứng trong Golang. So sánh các kiểu tương ứng trong <stdint.h> được trình bày trong Bảng 2-2.

| Kiểu trong C | Kiểu trong CGO | Kiểu trong Go |
| ------------ | -------------- | ------------- |
| int8_t       | C.int8_t       | int8          |
| uint8_t      | C.uint8_t      | uint8         |
| int16_t      | C.int16_t      | int16         |
| uint16_t     | C.uint16_t     | uint16        |
| int32_t      | C.int32_t      | int32         |
| uint32_t     | C.uint32_t     | uint32        |
| int64_t      | C.int64_t      | int64         |
| uint64_t     | C.uint64_t     | uint64        |

_Bảng 2-2 So sánh kiểu trong `stdint.h` _

Như đã đề cập trước đó, nếu kiểu trong C bao gồm nhiều từ khóa, nó không thể được sử dụng trực tiếp thông qua package "C" ảo (ví dụ: `unsigned short` không thể được truy cập trực tiếp `C.unsigned short`). Tuy nhiên, sau khi định nghĩa lại kiểu trong <stdint.h> bằng cách sử dụng `typedef`, chúng ta có thể truy cập tới kiểu gốc. Đối với các kiểu trong C phức tạp hơn thì nên sử dụng `typedef` để đặt lại tên cho nó, thuận tiện hơn cho việc truy cập trong CGO.

## 2.3.2 Go Strings và Slices

Trong file header `_cgo_export.h` được tạo ra bởi CGO, kiểu trong C tương ứng cũng được tạo cho các kiểu của Go như string, slice, dictionary, interface và pipe:

```go
typedef struct { const char *p; GoInt n; } GoString;
typedef void *GoMap;
typedef void *GoChan;
typedef struct { void *t; void *v; } GoInterface;
typedef struct { void *data; GoInt len; GoInt cap; } GoSlice;
```

Tuy nhiên, cần lưu ý rằng chỉ các string và slice là có giá trị sử dụng trong CGO, vì CGO tạo ra các phiên bản ngôn ngữ C cho một số hàm trong Go, vì vậy cả hai đều có thể gọi các hàm C trong Go, điều này được thực hiện lặp tức và CGO không cung cấp các hàm hỗ trợ liên quan cho các kiểu khác, đồng thời mô hình bộ nhớ dành riêng cho ngôn ngữ Go ngăn chúng ta duy trì các kiểu con trỏ tới bộ nhớ này quản lý bởi Go, vì vậy mà môi trường ngôn ngữ C của các kiểu đó không có giá trị sử dụng.

Trong hàm C đã export, chúng ta có thể trực tiếp sử dụng các string và slice trong Go. Giả sử bạn có hai hàm export sau:

```go
//export helloString
func helloString(s string) {}

//export helloSlice
func helloSlice(s []byte) {}
```

File header `_cgo_export.h` được tạo bởi CGO sẽ chứa khai báo hàm sau:

```c
extern void helloString(GoString p0);
extern void helloSlice(GoSlice p0);
```

Nhưng lưu ý rằng nếu bạn sử dụng kiểu `GoString` thì sẽ phụ thuộc vào file header `_cgo_export.h` và tập file này có output động.

Phiên bản Go1.10 thêm một chuỗi kiểu `_GoString_` định nghĩa trước, có thể làm giảm code có rủi ro phụ thuộc file header `_cgo_export.h`. Chúng ta có thể điều chỉnh khai báo ngôn ngữ C của hàm `helloString` thành:

```c
extern void helloString(_GoString_ p0);
```

## 2.3.3 Struct, Union, Enumerate

Các kiểu struct, Union và Enumerate của ngôn ngữ C không thể được nhúng dưới dạng thành phần ẩn danh vào struct của ngôn ngữ Go. Trong Go, chúng ta có thể truy cập các kiểu struct như `struct xxx` tương ứng là `C.struct_xxx` trong ngôn ngữ C. Bố cục bộ nhớ của struct tuân theo các quy tắc căn chỉnh (alignment) chung của ngôn ngữ C. Trong môi trường ngôn ngữ Go 32 bit, struct của C cũng tuân theo quy tắc căn chỉnh 32 bit và môi trường ngôn ngữ Go 64 bit tuân theo quy tắc căn chỉnh 64 bit. Đối với các struct có quy tắc căn chỉnh đặc biệt được chỉ định, chúng không thể được truy cập trong CGO.

Cách sử dụng struct đơn giản như sau:

```go
/*
struct A {
    int i;
    float f;
};
*/
import "C"
import "fmt"

func main() {
    var a C.struct_A
    fmt.Println(a.i)
    fmt.Println(a.f)
}
```

Nếu tên thành phần của struct tình cờ là một từ khóa trong  Go, bạn có thể truy cập nó bằng cách thêm một dấu gạch dưới ở đầu tên thành viên:

```go
/*
struct A {
    int type; // type là một từ khóa trong Golang
};
*/
import "C"
import "fmt"

func main() {
    var a C.struct_A
    fmt.Println(a._type) // _type tương ứng với type
}
```

Nhưng nếu có 2 thành phần: một thành phần được đặt tên theo từ khóa của Go và phần kia là trùng khi thêm vào dấu gạch dưới, thì các thành phần được đặt tên theo từ khóa ngôn ngữ Go sẽ không thể truy cập (bị chặn):

```go
/*
struct A {
    int   type;  // type là một từ khóa trong Go
    float _type; // chặn CGO truy cập type trên kia
};
*/
import "C"
import "fmt"

func main() {
    var a C.struct_A
    fmt.Println(a._type) // _type tương ứng với _type
}
```

Các thành phần tương ứng với trường bit (biến được định nghĩa với giá trị độ lớn cho sẵn) trong cấu trúc ngôn ngữ C không thể được truy cập bằng ngôn ngữ Go. Nếu bạn cần thao tác với các thành phần này, bạn cần xác định hàm hỗ trợ trong ngôn ngữ C.

Đối với các thành phần của mảng có độ dài bằng 0, các phần tử của mảng không thể truy cập trực tiếp trong Go, nhưng vẫn có thể truy cập phần bù vị trí (offset) của phần tử trong mảng có độ dài bằng 0 thông qua `unsafe.Offsetof(a.arr)`.

```go
/*
struct A {
    int   size: 10; // Trường bit không thể truy cập
    float arr[];    // Mạng có độ dài bằng 0 cũng không thể truy cập được
};
*/
import "C"
import "fmt"

func main() {
    var a C.struct_A
    fmt.Println(a.size) // Lỗi không thể truy cập trường bit
    fmt.Println(a.arr)  // Lỗi mảng có độ dài bằng 0
}
```

Trong ngôn ngữ C, chúng ta không thể truy cập trực tiếp vào kiểu struct được xác định bởi ngôn ngữ Go.

Đối với các kiểu union, chúng ta có thể truy cập các kiểu `union xxx` tương ứng là `C.union_xxx` trong ngôn ngữ C. Tuy nhiên, các kiểu union trong C không được hỗ trợ trong Go và chúng được chuyển đổi thành các mảng byte có kích thước tương ứng.

```go
/*
#include <stdint.h>

union B1 {
    int i;
    float f;
};

union B2 {
    int8_t i8;
    int64_t i64;
};
*/
import "C"
import "fmt"

func main() {
    var b1 C.union_B1;
    fmt.Printf("%T\n", b1) // [4]uint8

    var b2 C.union_B2;
    fmt.Printf("%T\n", b2) // [8]uint8
}
```

Nếu bạn cần thao tác biến kiểu lồng nhau trong C (union), thường có ba phương pháp: cách thứ nhất là xác định hàm hỗ trợ trong C, cách thứ hai là giải mã thủ công các thành phần thông qua "encoding/binary" của ngôn ngữ Go (không phải vấn đề big endian), thứ ba là sử dụng package `unsafe` để chuyển sang kiểu tương ứng (đây là cách tốt nhất để thực hiện). Sau đây cho thấy cách truy cập các thành viên kiểu union thông qua package `unsafe`:

```go
/*
#include <stdint.h>

union B {
    int i;
    float f;
};
*/
import "C"
import "fmt"

func main() {
    var b C.union_B;
    fmt.Println("b.i:", *(*C.int)(unsafe.Pointer(&b)))
    fmt.Println("b.f:", *(*C.float)(unsafe.Pointer(&b)))
}
```

Mặc dù truy cập bằng package `unsafe` là cách dễ nhất và tốt nhất về hiệu suất, nó có thể làm phức tạp vấn đề với các tình huống mà trong đó các kiểu union lồng nhau được xử lý. Đối với các kiểu này ta nên xử lý chúng bằng cách xác định các hàm hỗ trợ trong ngôn ngữ C.

Đối với các kiểu liệt kê (enum), chúng ta có thể truy cập các kiểu `enum xxx` tương ứng là `C.enum_xxx` trong C.

```go
/*
enum C {
    ONE,
    TWO,
};
*/
import "C"
import "fmt"

func main() {
    var c C.enum_C = C.TWO
    fmt.Println(c)
    fmt.Println(C.ONE)
    fmt.Println(C.TWO)
}
```

Trong ngôn ngữ C, kiểu `int` bên dưới kiểu liệt kê hỗ trợ giá trị âm. Chúng ta có thể truy cập trực tiếp các giá trị liệt kê được xác định bằng `C.ONE`, `C.TWO`, v.v.

## 2.3.4 Array, String và Slice

Trong C, biến mảng thực ra tương ứng với một con trỏ trỏ tới một phần bộ nhớ có độ dài cụ thể của một kiểu cụ thể, con trỏ này không thể được sửa đổi, khi truyền biến mảng vào một hàm, thực ra là truyền địa chỉ phần tử đầu tiên của mảng. Ở đây ta xem một độ dài nhất định của bộ nhớ là một mảng. Chuỗi trong C là một mảng kiểu char và độ dài của nó phải được xác định theo vị trí của ký tự NULL (đại diện kết thúc mảng). Không có kiểu slice trong ngôn ngữ C.

Trong Go, mảng là một kiểu giá trị và độ dài của mảng là một phần của kiểu mảng. Chuỗi trong Go tương ứng với một vùng nhớ "chỉ đọc" có độ dài nhất định. Slice trong Go là phiên bản đơn giản hơn của mảng động (dynamic array).

Chuyển đổi giữa Go và C với các kiểu array, string và slice có thể được đơn giản hóa thành chuyển đổi giữa Go slice và C pointer trỏ tới vùng nhớ có độ dài nhất định.

Package C ảo của CGO cung cấp tập các hàm sau để chuyển đổi hai chiều array và string giữa Go và C:

```go
// Go string to C string
// The C string is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be
// freed, such as by calling C.free (be sure to include stdlib.h
// if C.free is needed).
func C.CString(string) *C.char

// Go []byte slice to C array
// The C array is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be
// freed, such as by calling C.free (be sure to include stdlib.h
// if C.free is needed).
func C.CBytes([]byte) unsafe.Pointer

// C string to Go string
func C.GoString(*C.char) string

// C data with explicit length to Go string
func C.GoStringN(*C.char, C.int) string

// C data with explicit length to Go []byte
func C.GoBytes(unsafe.Pointer, C.int) []byte
```

Đối với chuỗi Go đầu vào của `C.CString`, nó sẽ được sao chép sang chuỗi định dạng ngôn ngữ C, chuỗi trả về được gán bởi hàm `malloc` của C và cần phải được release bằng `free` (của C) khi không sử dụng. Hàm `C.CBytes` và các hàm tương tự được sử dụng để nhân bản (clone) ra phiên bản C của một mảng byte từ slice byte của ngôn ngữ Go đầu vào. Mảng trả về tương tự cần được release vào thời điểm thích hợp. `C.GoString` được sử dụng để nhân bản ra từ chuỗi kết thúc NULL của ngôn ngữ C. `C.GoBytes` được sử dụng để nhân bản một slice byte của ngôn ngữ Go từ một mảng trong C.

Các hàm hỗ trợ này được chạy trong chế độ nhân bản. Khi string và slice của Go được chuyển đổi thành phiên bản trong C, bộ nhớ nhân bản được cấp phát bởi hàm `malloc` của C và cuối cùng có thể được giải phóng bằng `free`. Khi một chuỗi hoặc mảng trong C được chuyển đổi thành Go, bộ nhớ nhân bản được quản lý bởi ngôn ngữ Go. Với bộ hàm chuyển đổi này, bộ nhớ trước chuyển đổi và sau chuyển đổi vẫn ở trong vùng nhớ cục bộ tương ứng của chúng. Ưu điểm của chuyển đổi trong chế độ nhân bản là quản lý interface và bộ nhớ rất đơn giản. Nhược điểm là nhân bản cần phân bổ bộ nhớ mới và các hoạt động sao chép của nó sẽ dẫn nhiều đến chi phí phụ.

Các định nghĩa cho string và slice trong package `reflect`:

```go
type StringHeader struct {
    Data uintptr
    Len  int
}

type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

Nếu không muốn phân bổ bộ nhớ riêng, bạn có thể truy cập trực tiếp vào không gian bộ nhớ của C bằng Go:

```go
/*
#include <string.h>
char arr[10];
char *s = "Hello";
*/
import "C"
import (
    "reflect"
    "unsafe"
    "fmt"
)

func main() {
    // chuyển đổi bằng reflect.SliceHeader
    var arr0 []byte
    var arr0Hdr = (*reflect.SliceHeader)(unsafe.Pointer(&arr0))
    arr0Hdr.Data = uintptr(unsafe.Pointer(&C.arr[0]))
    arr0Hdr.Len = 10
    arr0Hdr.Cap = 10

    // chuyển đổi slice
    arr1 := (*[31]byte)(unsafe.Pointer(&C.arr[0]))[:10:10]


    var s0 string
    var s0Hdr = (*reflect.StringHeader)(unsafe.Pointer(&s0))
    s0Hdr.Data = uintptr(unsafe.Pointer(C.s))
    s0Hdr.Len = int(C.strlen(C.s))

    sLen := int(C.strlen(C.s))
    s1 := string((*[31]byte)(unsafe.Pointer(C.s))[:sLen:sLen])

    fmt.Println("arr1: ", arr1)
    fmt.Println("s1: ", s1)
}
```

Vì chuỗi trong Go là chuỗi chỉ đọc, người dùng cần đảm bảo rằng nội dung của chuỗi C bên dưới sẽ không thay đổi trong quá trình sử dụng chuỗi đó trong Go và bộ nhớ sẽ không được giải phóng trước.

Trong CGO, phiên bản ngôn ngữ C của struct tương ứng với struct string và slice trên:

```go
typedef struct { const char *p; GoInt n; } GoString;
typedef struct { void *data; GoInt len; GoInt cap; } GoSlice;
```

Trong C có thể dùng `GoString` và `GoSlice` để truy cập chuỗi và slice trong Go. Nếu là một kiểu mảng trong Go, bạn có thể chuyển đổi mảng thành một slice và sau đó chuyển đổi nó. Nếu không gian bộ nhớ bên dưới tương ứng với một chuỗi hoặc slice được quản lý bởi runtime của Go thì đối tượng bộ nhớ Go có thể được lưu trong một thời gian dài trong ngôn ngữ C.

Chi tiết về mô hình bộ nhớ CGO sẽ được thảo luận kĩ hơn trong các chương sau.

## 2.3.5 Chuyển đổi giữa các con trỏ

Trong ngôn ngữ C, các kiểu con trỏ khác nhau có thể được chuyển đổi tường minh hoặc ngầm định. Nếu là ngầm định, nó sẽ chỉ đưa ra một số thông tin cảnh báo tại thời điểm biên dịch. Nhưng ngôn ngữ Go rất nghiêm ngặt đối với các kiểu chuyển đổi khác nhau và mọi thông báo cảnh báo có thể xuất hiện trong ngôn ngữ C có thể sẽ là error trong ngôn ngữ Go. Con trỏ là linh hồn của ngôn ngữ C và việc chuyển đổi tự do giữa các con trỏ cũng là vấn đề quan trọng đầu tiên thường được giải quyết trong code CGO.

Trong ngôn ngữ Go, hai con trỏ hoàn toàn giống nhau về kiểu và có thể được sử dụng trực tiếp mà không cần chuyển đổi. Nếu một kiểu con trỏ được xây dựng dựa trên một kiểu con trỏ khác, nói cách khác, hai con trỏ bên dưới là các con trỏ có cùng cấu trúc, thì chúng ta có thể chuyển đổi giữa các con trỏ bằng cú pháp cast trực tiếp. Tuy nhiên, CGO thường phải đối phó với việc chuyển đổi giữa hai kiểu con trỏ hoàn toàn khác nhau. Về nguyên tắc, thao tác này bị nghiêm cấm trong code Go thuần.

Một trong những mục đích của CGO là phá vỡ sự cấm đoán nói trên và khôi phục các thao tác chuyển đổi con trỏ tự do mà ngôn ngữ C nên có. Đoạn code sau trình bày cách chuyển đổi một con trỏ kiểu X thành một con trỏ kiểu Y:

```go
var p *X
var q *Y

q = (*Y)(unsafe.Pointer(p)) // *X => *Y
p = (*X)(unsafe.Pointer(q)) // *Y => *X
```

Để chuyển đổi con trỏ kiểu X thành con trỏ kiểu Y, chúng ta cần hiện thực hàm `unsafe.Pointer` chuyển đổi giữa các kiểu con trỏ khác nhau như một kiểu cầu nối trung gian. Kiểu con trỏ `unsafe.Pointer` tương tự với ngôn ngữ C với con trỏ `void*`.

Sau đây là sơ đồ quá trình chuyển đổi giữa các con trỏ:

<div align="center">
<img src="../images/ch2-1-x-ptr-to-y-ptr.uml.png">
<br/>
<span align="center"><i>Con trỏ kiểu X thành con trỏ kiểu Y</i></span>
</div>
<br/>

Bất kỳ kiểu con trỏ nào cũng có thể được chuyển sang kiểu con trỏ `unsafe.Pointer` để bỏ đi thông tin kiểu ban đầu, sau đó gán lại một kiểu con trỏ mới để đạt được mục đích chuyển đổi.

## 2.3.6 Chuyển đổi giá trị và con trỏ

Việc chuyển đổi giữa các kiểu con trỏ khác nhau có vẻ phức tạp, nhưng nó tương đối đơn giản trong CGO. Trong ngôn ngữ C, ta thường gặp trường hợp con trỏ được biểu diễn bởi giá trị thông thường, điều này nghĩa là làm thế nào để hiện thực việc chuyển đổi giá trị và con trỏ cũng là một vấn đề mà CGO cần phải đối mặt.

Để kiểm soát chặt chẽ việc sử dụng con trỏ, ngôn ngữ Go cấm chuyển đổi các kiểu số trực tiếp thành các kiểu con trỏ. Tuy nhiên, Go đã đặc biệt định nghĩa một kiểu uintptr cho các kiểu con trỏ `unsafe.Pointer`. Chúng ta có thể sử dụng uintptr làm trung gian để hiện thực các kiểu số thành các kiểu `unsafe.Pointer`. Kết hợp với các phương pháp đã đề cập trước đó, việc chuyển đổi các giá trị và con trỏ giờ đã có thể thực hiện.

Biểu đồ sau đây trình bày cách hiện thực chuyển đổi lẫn nhau của kiểu `int32` sang kiểu con trỏ `char*` là chuỗi trong ngôn ngữ C:

<div align="center">
	<img src="../images/ch2-2-int32-to-char-ptr.uml.png">
	<br/>
	<span align="center">
		<i>Int32 và char chuyển đổi con trỏ</i>
	</span>
</div>
<br/>

Việc chuyển đổi được chia thành nhiều giai đoạn và mục tiêu ở mỗi giai đoạn: đầu tiên là kiểu `int32` sang `uintptr`, sau đó là `uintptr` thành kiểu con trỏ `unsafe.Pointr` và cuối cùng là kiểu con trỏ `unsafe.Pointr` thành kiểu `*C.char`.

## 2.3.7 Chuyển đổi giữa kiểu slice

Mảng cũng là một loại con trỏ trong ngôn ngữ C, vì vậy việc chuyển đổi giữa hai kiểu mảng khác nhau về cơ bản tương tự như chuyển đổi giữa các con trỏ. Tuy nhiên, trong ngôn ngữ Go, slice tương ứng với một con trỏ tới một mảng (fat pointer), vì vậy chúng ta không thể chuyển đổi trực tiếp giữa các kiểu slice khác nhau.

Tuy nhiên, package `reflection` của ngôn ngữ Go đã cung cấp sẵn cấu trúc cơ bản của kiểu slice nhờ đó chuyển đổi slice có thể được hiện thực và kết hợp với kỹ thuật chuyển đổi con trỏ được thảo luận ở trên giữa các kiểu khác nhau:

```go
var p []X
var q []Y

pHdr := (*reflect.SliceHeader)(unsafe.Pointer(&p))
qHdr := (*reflect.SliceHeader)(unsafe.Pointer(&q))

pHdr.Data = qHdr.Data
pHdr.Len = qHdr.Len * unsafe.Sizeof(q[0]) / unsafe.Sizeof(p[0])
pHdr.Cap = qHdr.Cap * unsafe.Sizeof(q[0]) / unsafe.Sizeof(p[0])
```

Ý tưởng chuyển đổi giữa các kiểu slice khác nhau là trước tiên xây dựng một slice trống, sau đó điền vào slice đó với dữ liệu bên dưới của slice gốc. Nếu kiểu X và Y có kích thước khác nhau, bạn cần đặt lại thuộc tính Len và Cap. Cần lưu ý rằng nếu X hoặc Y là kiểu null, đoạn code trên có thể gây ra lỗi chia cho 0 và code thực tế cần được xử lý khi thích hợp.

Sau đây cho thấy luồng cụ thể của thao tác chuyển đổi giữa các slice:

<div align="center">
	<img src="../images/ch2-3-x-slice-to-y-slice.uml.png">
	<br/>
	<span align="center">
		<i>kiểu cắt X thành slice Y</i>
	</span>
</div>
<br/>

Đối với các tính năng thường được sử dụng trong CGO, tác giả package <github.com/chai2010/cgo>, đã cung cấp các chức năng chuyển đổi cơ bản. Để biết thêm chi tiết hãy tham khảo code hiện thực.
