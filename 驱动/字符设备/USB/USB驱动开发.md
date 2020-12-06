### 1、前言  

我们一般所说的usb驱动开发是指usb_driver的开发而不是usb_device_driver的开发

### 2、usb接口驱动和usb设备驱动  

> 接口驱动：usb_driver（我们一般的驱动开发）

1、一个驱动往往可以支持多个接口   

```c
struct usb_driver {
    const char *name;
    int (*probe) (struct usb_interface *intf,const struct usb_device_id *id);
    void (*disconnect) (struct usb_interface *intf);
    int (*unlocked_ioctl) (struct usb_interface *intf, unsigned int code, void *buf);
    int (*suspend) (struct usb_interface *intf, pm_message_t message);
    int (*resume) (struct usb_interface *intf);
    int (*reset_resume)(struct usb_interface *intf);
    int (*pre_reset)(struct usb_interface *intf);
    int (*post_reset)(struct usb_interface *intf);
    const struct usb_device_id *id_table;
    struct usb_dynids dynids;
    struct usbdrv_wrap drvwrap;
    unsigned int no_dynamic_id:1;
    unsigned int supports_autosuspend:1;
    unsigned int disable_hub_initiated_lpm:1;
    unsigned int soft_unbind:1;
};
```

> 设备驱动：usb_device_driver  

1、这里的usb设备驱动指的是所有的usb设备的驱动
```c
struct usb_device_driver {
    const char *name;
    int (*probe) (struct usb_device *udev);
    void (*disconnect) (struct usb_device *udev);
    int (*suspend) (struct usb_device *udev, pm_message_t message);
    int (*resume) (struct usb_device *udev, pm_message_t message);
    struct usbdrv_wrap drvwrap;
    unsigned int supports_autosuspend:1;
};
```

