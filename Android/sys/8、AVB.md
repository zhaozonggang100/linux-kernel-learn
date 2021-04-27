### 1、ref

https://evilpan.com/2020/11/14/android-secure-boot/

### 2、android Q上使用avbtool信息

```bash
$ avbtool info_image --image out/target/product/xxx/vbmeta.img 
Minimum libavb version:   1.0
Header Block:             256 bytes
Authentication Block:     576 bytes
Auxiliary Block:          2880 bytes
Algorithm:                SHA256_RSA4096
Rollback Index:           0
Flags:                    0
Release String:           'avbtool 1.1.0'
Descriptors:
    Chain Partition descriptor:
      Partition Name:          vbmeta_system
      Rollback Index Location: 2
      Public key (sha1):       cdbb77177f731920bbe0a0f94f84d9038ae0617d
    Prop: com.android.build.boot.os_version -> '10'
    Prop: com.android.build.boot.security_patch -> '2020-08-05'
    Prop: com.android.build.vendor.os_version -> '10'
    Prop: com.android.build.vendor.security_patch -> '2020-08-05'
    Hash descriptor:
      Image Size:            50462720 bytes
      Hash Algorithm:        sha256
      Partition Name:        boot
      Salt:                  ddf243797fa7866083a95061d8b46c10d6e2ab9f65da9dc26bfa32104ff1ab82
      Digest:                abc6797e2f87fdcc09e8d8acce2f6bee9f0057c9d0b2b233cb596487caa5b218
      Flags:                 0
    Hash descriptor:
      Image Size:            744877 bytes
      Hash Algorithm:        sha256
      Partition Name:        dtbo
      Salt:                  83b00c7d785b05fdcef0566fe96d25eebfad186b15cef41e7c7e6e38d464941b
      Digest:                5f8a861a9c3942d9d8bf7191dc46dd60606b82972be47efa373e2b7f770e5bab
      Flags:                 0
    Hash descriptor:
      Image Size:            56930304 bytes
      Hash Algorithm:        sha256
      Partition Name:        recovery
      Salt:                  f00fbced9cdad81009e4472ec9743013e31e90505ee98a6eb318cb4f9e15a97a
      Digest:                353f6fdc53744764cf53acef1cb95f11f3544c96d9f416b4218c70031b8923cf
      Flags:                 0
    Hashtree descriptor:
      Version of dm-verity:  1
      Image Size:            521314304 bytes
      Tree Offset:           521314304
      Tree Size:             4112384 bytes
      Data Block Size:       4096 bytes
      Hash Block Size:       4096 bytes
      FEC num roots:         2
      FEC offset:            525426688
      FEC size:              4161536 bytes
      Hash Algorithm:        sha1
      Partition Name:        vendor
      Salt:                  fe652ae1ee261e83803fc9b076808203f6255b7d
      Root Digest:           0cbb4e767cc6485302848228e1697f3c38119599
      Flags:                 0

$ avbtool info_image --image out/target/product/xxx/vbmeta_system.im
Minimum libavb version:   1.0
Header Block:             256 bytes
Authentication Block:     320 bytes
Auxiliary Block:          960 bytes
Algorithm:                SHA256_RSA2048
Rollback Index:           1596585600
Flags:                    0
Release String:           'avbtool 1.1.0'
Descriptors:
    Prop: com.android.build.system.os_version -> '10'
    Prop: com.android.build.system.security_patch -> '2020-08-05'
    Hashtree descriptor:
      Version of dm-verity:  1
      Image Size:            1653833728 bytes
      Tree Offset:           1653833728
      Tree Size:             13029376 bytes
      Data Block Size:       4096 bytes
      Hash Block Size:       4096 bytes
      FEC num roots:         2
      FEC offset:            1666863104
      FEC size:              13180928 bytes
      Hash Algorithm:        sha1
      Partition Name:        system
      Salt:                  a5de187b9ca5759f2f40f728c07c9518151da91a
      Root Digest:           b7cea353bdb002025c63abcfa02225c8b58a7d36
      Flags:                 0
```