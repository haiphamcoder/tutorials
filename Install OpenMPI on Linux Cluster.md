# Hướng dẫn cài đặt OpenMPI trên Linux Cluster

Để thiết lập một cụm trong môi trường cục bộ, các phiên bản OpenMPI giống nhau phải được cài đặt sẵn trong mọi hệ thống.

## Điều kiện tiên quyết

- Hệ điều hành: Ubuntu 20.04.6 Desktop LTS x86_64

- MPI: <https://www.open-mpi.org/software/>

    (Hướng dẫn này tuân theo OpenMPI v5.0.2)

## Các bước để tạo cụm MPI

### Bước 1: Tạo người dùng mới

Chúng ta có thể vận hành cụm bằng cách sử dụng user hiện có. Nhưng tốt hơn hết là tạo một người dùng mới để mọi việc đơn giản hơn. Tạo tài khoản người dùng mới có cùng tên người dùng trong tất cả các máy để đơn giản hóa mọi việc.

Ta thực hiện trên tất cả các máy như sau:

- Để thêm người dùng mới:

  ```bash
  sudo adduser mpiuser
  ```

  (Chú ý: Đặt mật khẩu giống nhau cho tài khoản `mpiuser` trên tất cả các máy)

- Biến tài khoản `mpiuser` thành `sudoer`:

  ```bash
  sudo usermod -aG sudo mpiuser 
  ```

- Logout khỏi user hiện tại và Login với user `mpiuser` vừa tạo.

### Bước 2: Cài đặt bộ công cụ mạng net-tools cho Ubuntu

- Cài đặt net-tools:

  ```bash
  sudo apt install net-tools
  ```

- Kiểm tra địa chỉ ip trên mỗi máy

  ```bash
  ifconfig
  ```

  Kết quả trông như sau (còn tùy thuộc vào kết nối mạng bằng wifi hay cáp LAN):

  ```output
  ...
  wlp0s20f3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          inet 192.168.76.240  netmask 255.255.240.0  broadcast 192.168.79.255
          inet6 fe80::f91f:985:87e4:224f  prefixlen 64  scopeid 0x20<link>
      ...
  ```

  Giá trị **192.168.76.240** chính là địa chỉ ip của máy đó.

- Giả sử, cluster của ta gồm 4 node. Ta kiểm tra được ip lần lượt là:

  ```output
  Máy A: 192.168.76.240
  Máy B: 192.168.76.114
  Máy C: 192.168.76.80
  Máy D: 192.168.76.34
  ```

- Chọn máy A làm master node, các máy còn lại sẽ là slave node

### Bước 3: Configure hosts file

Chúng ta sẽ liên lạc giữa các máy tính và chúng ta không muốn nhập địa chỉ IP thường xuyên. Thay vào đó, chúng ta có thể đặt tên cho các nút khác nhau trong mạng mà chúng ta muốn liên lạc. Tệp **/etc/hosts** được hệ điều hành thiết bị sử dụng để ánh xạ tên host thành địa chỉ IP.

- Trên máy A (master node), ta thực hiện:

  ```bash
  sudo nano /etc/hosts
  ```

  Sau đó thêm IP slave node vào tệp này như dưới đây và lưu lại:

  Chú ý: các dòng khác thì comment hết

  ```output
  127.0.0.1       localhost
  192.168.76.114  slave1
  192.168.76.80   slave2
  192.168.76.34   slave3
  ```

- Trên máy B (slave node), ta thực hiện:

  ```bash
  sudo nano /etc/hosts
  ```

  Sau đó thêm IP master node vào tệp này như dưới đây và lưu lại:

  Chú ý: các dòng khác thì comment hết

  ```output
  127.0.0.1       localhost
  192.168.76.240  master
  ```

- Các máy C, D làm tương tự như với máy B.

### Bước 4: Thiết lập SSH

Máy sẽ giao tiếp qua mạng thông qua SSH và chia sẻ dữ liệu qua NFS. Thực hiện theo quy trình dưới đây cho cả master node và slave node.

- Để cài đặt ssh trong hệ thống.

  ```bash
  sudo apt install openssh-server openssh-client
  ```

- Tạo khóa RSA để xác thực SSH, ta dùng lệnh dưới đây:

  ```bash
  ssh-keygen -t rsa
  ```

  ```output
  Generating public/private rsa key pair.
  Enter file in which to save the key (/home/haipn/.ssh/id_rsa):
  ```

  Nhấn `Enter` để xác nhận dùng đường dẫn mặc định `/home/mpiuser/.ssh/id_rsa`

  ```output
  Enter passphrase (empty for no passphrase):
  ```

  Nhập passphrase ở đây giống nhau cho tất cả các máy.

  Lúc này hệ thống sẽ sinh ra 2 file `id_rsa` và `id_rsa.pub` trong thư mục `/home/mpiuser/.ssh`

- Điều hướng đến thư mục ~/.ssh:
  
  ```bash
  cd ~/.ssh
  ```

  Sau đó thực hiện:

  ```bash
  cat id_rsa.pub >> authorized_keys
  ```

- Copy SSH Public Key giữa các node
  
  - Đối với master node, ta thực hiện lần lượt các lệnh sau:

    ```bash
    ssh-copy-id slave1
    ssh-copy-id slave2
    ssh-copy-id slave3
    ```

    Chú ý với mỗi lệnh, ta đều chọn `yes` và nhập mật khẩu đã tạo (giống nhau) cho mọi tài khoản `mpiuser` mà ta đã làm ở bước 1.

  - Đối với các slave node (máy B, C và D), ta thực hiện lệnh sau:

    ```bash
    ssh-copy-id master
    ```

### Bước 5: Thiết lập NFS

Chúng ta chia sẻ một thư mục thông qua NFS trong `master node` mà `slave node` sẽ gắn kết để trao đổi dữ liệu.

- Cài đặt các gói cần thiết:

  - Với `master node`, ta thực hiện:

    ```bash
    sudo apt install nfs-server
    ```

  - Với `slave node`, ta thực hiện:
  
    ```bash
    sudo apt install nfs-client
    ```

- Tạo thư mục `SharedFolder` để trao đổi dữ liệu. Trên tất cả các máy, ta thực hiện:
  
  ```bash
  mkdir -p /home/mpiuser/Desktop/SharedFolder
  sudo chown nobody:nogroup /home/mpiuser/Desktop/SharedFolder
  sudo chmod 777 /home/mpiuser/Desktop/SharedFolder
  ```

- Trên `master node`, ta thực hiện như dưới đây để cấp quyền chia sẻ thư mục `SharedFolder` cho các `slave node`:

  - Chỉnh sửa file `/etc/exports`:

    ```bash
    sudo nano /etc/exports
    ```

    Thêm vào các dòng sau:

    ```output
    /home/mpiuser/Desktop/SharedFolder 192.168.76.114(rw,sync,no_root_squash,no_subtree_check)
    /home/mpiuser/Desktop/SharedFolder 192.168.76.80(rw,sync,no_root_squash,no_subtree_check)
    /home/mpiuser/Desktop/SharedFolder 192.168.76.34(rw,sync,no_root_squash,no_subtree_check)
    ```

    Sau đó, nhấn `Ctrl+O` -> `Enter` để lưu lại. `Ctrl+X` để thoát.

  - Tiếp đến, ta dùng lệnh sau:

    ```bash
    sudo exportfs -a
    ```
  
  - Khởi động lại máy chủ NFS

    ```bash
    sudo service nfs-server restart
    ```

- Sau khi thực hiện xong các bước trên, ta cần gắn kết các thư mục `SharedFolder` trên `master node` với `slave node`.

  Trên `slave node`, ta thực hiện:

  ```bash
  sudo mount master:/home/mpiuser/Desktop/SharedFolder /home/mpiuser/Desktop/SharedFolder
  ```

  Sau đó, ta kiểm tra bằng cách thêm một file bất kỳ vào vào thư mục `SharedFolder` trên `master node`, và reload lại thư mục `SharedFolder` trên `slave node`, nếu thấy dữ liệu đã được đồng bộ thì tức là ta đã thiết lập NFS thành công.

  Ta có thể dùng lệnh sau để kiểm tra các thư mục được gắn kết:

  ```bash
  df -h
  ```

  Chú ý: Mỗi lần khởi động lại hệ thống thì cần thực hiện `mount` lại.  

### Bước 6: Chạy chương trình MPI (Thực hiện trên Master Node)

- Trên `master node`, tải file chương trình [tại đây](/prime_mpi.c) và chuyển file này vào thư mục `SharedFolder` trên `master node`

- Điều hướng đến thư mục `SharedFolder`:

  ```bash
  cd /home/mpiuser/Desktop/SharedFolder
  ```

- Biên dịch file chương trình:

  ```bash
  mpicc prime_mpi.c -o prime_mpi
  ```

- Chạy chương trình:

  - Nếu ta chỉ muốn chạy trên `master node`, ta thực hiện lệnh sau:

    ```bash
    mpirun -np 2 ./prime_mpi
    ```

    Trong đó, `-np` là chỉ số tiến trình

  - Để chạy trong Cluster, ta thực hiện:

    ```bash
    mpirun --hostfile /etc/hosts -np 6 ./prime_mpi
    ```
