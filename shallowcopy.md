#  ✍️ Shallow Copy trong C++: Cách triển khai shallow copy trong C++

**Tác giả:** MaiTrungKien  
**Ngày:** 2025-08-01  
**Tags:** C++, Shallow Copy     
**Mức độ:** Trung – Cao

# ✍️ Trong bài viết này chúng ta sẽ đề cập đến một lỗi nghiêm trọng với kỹ thuật shallow copy khi tài nguyên của một class được cấp phát động (dynamic allocation)

### 👉  Mặc định khi không triển khai lại phương thức copy constructor hoặc assignment operator thì sẽ là shallow copy. Để hiểu rõ shallow copy chúng ta thực hiện đoạn code sau.

* Bước 1: Tạo class Camera, class shallowcopy. Trong class shallowcopy có tài nguyên động là đối tượng của class `Camera`
```cpp 
// shallowcopy.h
#pragma once
#include <iostream>

#ifndef OUT, END 
#define OUT std::cout 
#define END std::endl
#endif

class Camera {
    public:
        Camera(int _x, int _y): x(_x), y(_y){
            OUT << "[Camera] Calling Constructor" << END ;
        }
        ~Camera(){
            OUT <<"[Camera] Calling Destructor"<< END;
        }
        const int getW()const {return w;}
        const int getH()const {return h;}
        void setW(const int _w){w = _w;}
        void setH (const int _h){h=_h;}
    private:
        int w;
        int y;
};

class shallowcopy{
    public:
        shallowcopy(Camera* _cam_): cam(_cam){
            OUT << "[shallowcopy] Calling Constructor" <<END;
        }
        ~shallowcopy(){
            OUT <<"[shallowcopy] Calling Destructor"<< END;
        }
        void setW(const int _w){
            cam->setW(_w);
        }
        void setH(const int _h){
            cam->setH(_h);
        }
        const int getW()const {return cam->getW();}
        const int getH()const {return cam->getH();}

    private:
    Camera *cam = nullptr;
};
```
   
* Bước 2: Triển khai shallowcopy

```cpp
// main.cpp
int main(){
    {
        Camera * cam = new cam(1,2);
        shallowcopy s1(cam);
        shallowcopy s2 = s1; // Gọi Copy Constructor mặc định

        // Thay đổi giá trị W hoăc H trong s1 sau đó kiểm tra giá trị W hoặc H trong s2 và ngược lại
        s1.setW(99);
        OUT << "Giá trị của W trong s1 là: " << s1.getW() << END;
        OUT << "Giá trị của W trong s2 khi thực hiện s1.setW(99): " << s2.getW() << END;
    }
   
    return 0;
}
```

```text
[Camera] Calling Constructor
[shallowcopy] Calling Constructor
[shallowcopy] Calling Copy Constructor
Giá trị của W trong s1 là: 99
Giá trị của W trong s2 khi thực hiện s1.setW(99): 99
[shallowcopy] Calling Destructor
```
* Có thể thấy được cả tài nguyên động `cam` của đối tượng s1 và đối tượng s2 cùng tham chiếu tới một vùng nhớ đã được cấp phát từ dòng code 
```cpp 
Camera * cam = new cam(1,2);
```

### 👉 Điều gì đang thực sự xảy ra ở ví dụ trên.   
- Ở đây chúng ta thấy khi thực hiện hàm `setW()` của s1 và s2 giá trị W của tài nguyên động cam thuộc đối tượng s1 và s2 đều có giá trị giống nhau
- Bản chất tài nguyên động `cam` của s1 và tài nguyên động cam của s2 cùng tham chiếu tới một vùng nhớ.
- Khi đối tượng s1, s2 `out of scope` tức là bị hủy (với cấp phát tĩnh) thì vùng nhớ của chúng sẽ được tự động thu hồi. Nhưng đối với đối tượng cam là tài nguyên động chúng ta chỉ thấy hàm `Constructor` được gọi, sau khi kết thúc chương trình hàm `Destructor` của tài nguyên động `cam`vẫn không được gọi `->` điều này gây ra memmory leak
- Vậy mấu chốt vấn đề chỉ cần thu hồi bộ nhớ đã cấp phát cho tài nguyên động trong hàm huỷ của class quản lý chúng

### 👉 Thu hồi bộ nhớ `(deallocate dynamic resource)` đã cấp phát cho tài nguyên động
```cpp
...
// shallowcopy.h
class shallowcopy{
    public:
        shallowcopy(Camera* _cam_): cam(_cam){
            OUT << "[shallowcopy] Calling Constructor" <<END;
        }
        ~shallowcopy(){
            OUT <<"[shallowcopy] Calling Destructor"<< END;
            if(cam){
                delete cam;
                cam = nullptr;
            }
        }
        void setW(const int _w){
            cam->setW(_w);
        }
        void setH(const int _h){
            cam->setH(_h);
        }
        const int getW()const {return cam->getW();}
        const int getH()const {return cam->getH();}

    private:
    Camera *cam = nullptr;
};

```
- Mặc dù chúng ta đã thu hồi bộ nhớ cấp phát cho tài nguyên động `cam`. Nhưng rõ ràng cả 2 biến `cam` trong 2 đối tượng của class shallowcopy cùng tham chiếu tới một vùng nhớ dẫn tới hiện tượng `double free` hoặc `unbehavior` Điều này gây ra việc Crash chương trình, mặc dù chúng ra đã delete và đặt là `nullptr`
- Mấu chốt vấn đề là làm sao để biết tài nguyên động đã được deallocation, nếu đã được deallocation rồi thì chúng ra sẽ không thực hiện deallocation nữa.

### 👉 `std::vector<>` để quan sát số lượng tài nguyên động, tránh trường hợp double free hoặc unbehavior

```cpp
// shallowcopy.h
...
#include <vector>
#include <algorithm>

static std:vector<Camera*> vectorcam;
class shallowcopy{
    public:
        shallowcopy(Camera* _cam_): cam(_cam){
            auto it = std::find(vectorcam..begin(), vectorcam.end(),this->cam = _cam);
            if(it == vectorcam.end()){
                vectorcam.push_back(_cam);
            }
            OUT << "[shallowcopy] Calling Constructor" <<END;
        }
        ~shallowcopy(){
            OUT <<"[shallowcopy] Calling Destructor"<< END;
            auto it = std::find(vectorcam..begin(), vectorcam.end(),this->cam = _cam);
            if(it != vectorcam.end()){
                delete *it;
                vectorcam.erase(it);
            }
        }
        void setW(const int _w){
            cam->setW(_w);
        }
        void setH(const int _h){
            cam->setH(_h);
        }
        const int getW()const {return cam->getW();}
        const int getH()const {return cam->getH();}

    private:
    Camera *cam = nullptr;
};
```
### 👉 Trong ví dụ này nếu sử erase() trước khi delete thì sẽ gây ra lỗi, vì sao ?
* Bản chất của `std::vector<T>` lưu phần tử liên tiếp trong bộ nhớ vì vậy khi xóa một phần tử ở vị trí bất kỳ, các phần từ phía sau sẽ phải dịch sang bên trái (shift left) để lấp vào chỗ trống. Việc dữ liệu bị di dời hoặc ghi đè như vậy, các iterator trỏ vào vùng từ phần tử bị xóa trở đi không còn đảm bảo trỏ tới đúng phần tử cũ nữa (iterator invalid)
```cpp
vector.erase(it);
delete *it; // Crash chương trình
```
* Cách viết đúng
```cpp
delete *it;
vector.erase(it);
```

