**refs**：

https://www.cnblogs.com/cobbliu/articles/11898787.html
https://www.cnblogs.com/cherishui/p/3878678.html

---

### 1、info

scsi子系统由scsi底层、scsi中间层、scsi上层三部分构成

- 1、scsi底层

```
为底层真实的HBA（host bus adapter）提供接口（如给ufs分配HBA结构等）
```

- 2、scsi中间层

- 3、scsi上层  

```
处理通用块设备层提交到scsi子系统的io request（mmc子系统也有类似功能）
```