​	一般我们初学的时候，都会有如下代码，这和我们学习的设备驱动模型大不一样。

```
#define DEV_NAME "EmbedCharDev"
#define DEV_CNT (1)
#define BUFF_SIZE 128
//定义字符设备的设备号
static dev_t devno;
//定义字符设备结构体chr_dev
static struct cdev chr_dev;
static int __init chrdev_init(void)
{
   int ret = 0;
   printk("chrdev init\n");
   //第一步
   //采用动态分配的方式，获取设备编号，次设备号为0，
   //设备名称为EmbedCharDev，可通过命令cat /proc/devices查看
   //DEV_CNT为1，当前只申请一个设备编号
   ret = alloc_chrdev_region(&devno, 0, DEV_CNT, DEV_NAME);
   if (ret < 0) {
      printk("fail to alloc devno\n");
      goto alloc_err;
   }
   //第二步
   //关联字符设备结构体cdev与文件操作结构体file_operations
   cdev_init(&chr_dev, &chr_dev_fops);
   //第三步
   //添加设备至cdev_map散列表中
   ret = cdev_add(&chr_dev, devno, DEV_CNT);
   if (ret < 0) {
      printk("fail to add cdev\n");
      goto add_err;
   }
   return 0;

add_err:
   //添加设备失败时，需要注销设备号
   unregister_chrdev_region(devno, DEV_CNT);
alloc_err:
   return ret;
}
module_init(chrdev_init);
```





cdev并不会在sysfs中创建，需要手动添加。

```
rc = alloc_chrdev_region(&synx_dev->dev, 0, 1, SYNX_DEVICE_NAME);
if (rc < 0) {
    pr_err("region allocation failed\n");
    goto alloc_fail;
}

cdev_init(&synx_dev->cdev, &synx_fops);
synx_dev->cdev.owner = THIS_MODULE;
rc = cdev_add(&synx_dev->cdev, synx_dev->dev, 1);
if (rc < 0) {
    pr_err("device registation failed\n");
    goto reg_fail;
}
```





https://www.cnblogs.com/schips/p/linux_device_model_and_cdev_miscdev.html

https://blog.csdn.net/rikeyone/article/details/104019101





