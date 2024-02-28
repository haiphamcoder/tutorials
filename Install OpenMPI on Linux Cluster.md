# Hướng dẫn cài đặt OpenMPI trên Linux Cluster

Để thiết lập một cụm trong môi trường cục bộ, các phiên bản OpenMPI giống nhau phải được cài đặt sẵn trong mọi hệ thống.

## Điều kiện tiên quyết

- Hệ điều hành: Ubuntu 20.04.6 LTS x86_64

- MPI: <https://www.open-mpi.org/software/>

    (Hướng dẫn này tuân theo OpenMPI v5.0.2)

## Các bước để tạo cụm MPI

### Bước 1: Cài đặt bộ công cụ mạng net-tools cho Ubuntu

```bash
sudo apt install net-tools
```

Kiểm tra địa chỉ ip trên mối máy

```bash
ifconfig
```

Kết quả trông như sau:

```output
...
wlp0s20f3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.76.240  netmask 255.255.240.0  broadcast 192.168.79.255
        inet6 fe80::f91f:985:87e4:224f  prefixlen 64  scopeid 0x20<link>
    ...
```

Giá trị **192.168.76.240** chính là địa chỉ ip của máy đó.

Giả sử, cluster của ta gồm 4 node. Ta kiểm tra được ip lần lượt là:

```output
Máy A: 192.168.76.240
Máy B: 192.168.76.114
Máy C: 192.168.76.80
Máy D: 192.168.76.34
```

Chọn máy A làm master node, các máy còn lại sẽ là slave node

### Bước 2: Configure hosts file

Chúng ta sẽ liên lạc giữa các máy tính và chúng ta không muốn nhập địa chỉ IP thường xuyên. Thay vào đó, chúng ta có thể đặt tên cho các nút khác nhau trong mạng mà chúng ta muốn liên lạc. Tệp **/etc/hosts** được hệ điều hành thiết bị sử dụng để ánh xạ tên host thành địa chỉ IP.

Trên máy A (master node), ta thực hiện:

```bash
sudo nano /etc/hosts
```

Sau đó thêm IP master node và IP slave node vào tệp này như dưới đây và lưu lại:

Chú ý: các dòng khác thì comment hết

```output
127.0.0.1       localhost
192.168.76.240  master
192.168.76.114  slave1
192.168.76.80   slave2
192.168.76.34   slave3
```

Trên máy B (slave node), ta thực hiện:

```bash
sudo nano /etc/hosts
```

Sau đó thêm IP master node và IP slave node vào tệp này như dưới đây và lưu lại:

Chú ý: các dòng khác thì comment hết

```output
127.0.0.1       localhost
192.168.76.240  master
192.168.76.114  slave1
```

Các máy C, D làm tương tự như với máy B.

### Bước 3: Tạo người dùng mới

Chúng ta có thể vận hành cụm bằng cách sử dụng  user hiện có. Tốt hơn là tạo một người dùng mới để mọi việc đơn giản hơn. Tạo tài khoản người dùng mới có cùng tên người dùng trong tất cả các máy để đơn giản hóa mọi việc.

Ta thực hiện trên tất cả các máy như sau:

- Để thêm người dùng mới:

  ```bash
  sudo adduser mpiuser
  ```

- Biến mpiuser thành sudoer:

  ```bash
  sudo usermod -aG sudo mpiuser 
  ```

### Bước 4: Thiết lập SSH

Máy sẽ giao tiếp qua mạng thông qua SSH và chia sẻ dữ liệu qua NFS. Thực hiện theo quy trình dưới đây cho cả master node và slave node.

- Để cài đặt ssh trong hệ thống.

  ```bash
  sudo apt install openssh-server openssh-client
  ```

- Đăng nhập vào người dùng mới được tạo:

  ```bash
  su - mpiuser
  ```

- Điều hướng đến thư mục ~/.ssh:
  
  ```bash
  cd ~/.ssh
  ```
  
  Nếu đường dẫn không tồn tại, thì dùng lệnh sau để tạo:

  ```bash
  mkdir -p ~/.ssh
  ```

- Tạo khóa RSA để xác thực SSH:

  ```bash
  ssh-keygen -t rsa
  ```

  - Đối với master node, ta nhập lại đường dẫn lưu file như sau:

    ```output
    /home/mpiuser/.ssh/id_rsa_master
    ```

  - Đối với slave node (máy B), ta nhập lại đường dẫn lưu file như sau:

    ```output
    /home/mpiuser/.ssh/id_rsa_slave1
    ```

  - Đối với slave node C và D, ta làm tương tự như máy B, chỉ thay tên file lần lượt thành id_rsa_slave2 và id_rsa_slave3

- Copy file **/home/mpiuser/.ssh/id_rsa_master.pub** trong máy A (master node) sang các máy B, C, D (slave node)

- Copy file **/home/mpiuser/.ssh/id_rsa_slave1.pub** trong máy B (slave node) sang thư mục **/home/mpiuser/.ssh** trong máy A (master node). Tương tự với các máy C và D.



Bây giờ bạn có thể kết nối với các nút công nhân mà không cần nhập mật khẩu

nhân viên ssh2

Trong các nút công nhân sử dụng

trình quản lý ssh-copy-id

Bước 4: Thiết lập NFS

Chúng tôi chia sẻ một thư mục thông qua NFS trong trình quản lý mà nhân viên gắn kết để trao đổi dữ liệu.

NFS-Server cho nút chính:

Cài đặt các gói cần thiết bằng cách

    sudo apt-get cài đặt nfs-kernel-server

Chúng ta cần tạo một thư mục mà chúng ta sẽ chia sẻ trên mạng. Trong trường hợp của chúng tôi, chúng tôi đã sử dụng “đám mây”. Để xuất thư mục đám mây, chúng ta cần tạo một mục trong /etc/exports

sudo nano /etc/exports

Thêm vào

    /home/mpiuser/cloud *(rw,sync,no_root_squash,no_subtree_check)

Thay vì *, chúng tôi có thể cung cấp cụ thể địa chỉ IP mà chúng tôi muốn chia sẻ thư mục này hoặc chúng tôi có thể sử dụng*.

Ví dụ:

    /home/mpiuser/cloud 172.20.36.121(rw,sync,no_root_squash,no_subtree_check)

Sau khi nhập xong, hãy chạy như sau.

$ xuất khẩu -a

Chạy lệnh trên mỗi khi có bất kỳ thay đổi nào được thực hiện đối với /etc/exports.

Sử dụng Sudo importfs -a nếu câu lệnh trên không hoạt động.

Nếu được yêu cầu, hãy khởi động lại máy chủ NFS

    sudo dịch vụ nfs-kernel-server khởi động lại

> NFS-worker cho các nút máy khách

Cài đặt các gói cần thiết

$ sudo apt-get cài đặt nfs-common

Tạo một thư mục trong máy của công nhân có cùng tên – “cloud”

đám mây $ mkdir

Và bây giờ, hãy gắn thư mục dùng chung như

    sudo mount -t nfs manager:/home/mpiuser/cloud ~/cloud

Để kiểm tra các thư mục được gắn kết,

$ df -h

Đây là cách nó sẽ hiển thị

Để gắn kết vĩnh viễn để bạn không phải gắn thư mục dùng chung theo cách thủ công mỗi khi khởi động lại hệ thống, bạn có thể tạo một mục trong bảng hệ thống tệp của mình – tức là tệp /etc/fstab như thế này:

$ nano /etc/fstab

Thêm vào

# THIẾT LẬP CỤM MPI
 người quản lý:/home/mpiuser/cloud /home/mpiuser/cloud nfs

Bước 5: Chạy chương trình MPI

Điều hướng đến thư mục chia sẻ NFS (“đám mây” trong trường hợp của chúng tôi) và tạo các tệp ở đó [hoặc chúng tôi có thể chỉ dán các tệp đầu ra). Để biên dịch mã, giả sử tên của nó là mpi_hello.c, chúng ta sẽ phải biên dịch mã theo cách dưới đây để tạo ra một mpi_hello có thể thực thi được.

$ mpicc -o mpi_hello mpi_hello.c

Để chỉ chạy nó trong máy chính, chúng tôi làm

$ mpirun -np 2 ./mpi_helloBsend

np – Số tiến trình = 2

Để chạy mã trong một cụm

$ mpirun -hostfile my_host ./mpi_hello

Ở đây, tệp my_host xác định Địa chỉ IP và số lượng tiến trình sẽ được chạy.

Tệp máy chủ mẫu:

vị trí người quản lý=4 max_slots=40
 vị trí công nhân1=4 max_slots=40
 worker2 max_slots=40
 vị trí worker3=4 max_slots=40

Ngoài ra,

$ mpirun -np 5 -hosts worker,localhost ./mpi_hello

Lưu ý : Tên máy chủ cũng có thể được thay thế bằng địa chỉ IP.
