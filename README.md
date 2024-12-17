#Lektion 2 - Linux Kernel Interface kode:

## light.c:

```c

#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>

int main()
{

int fd, ret;

char on = '1';

char off = '0';





fd = open("/sys/class/gpio/gpio26/value",O_RDWR);

if(fd == -1){

printf("Fejl ved åbning af GPIO\n");

return -1;
}

while(1)
{

write(fd, &on, 1);

sleep(1);

write(fd, &off, 1);
sleep(1);

}
close(fd);
return ret;



}
```

## i2c.c:

```c
#include <stdbool.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <stdlib.h>
int main(){
int fd;
int ret = 0;
char rddata[4], str[16];
fd = open("/dev/i2c-1",O_RDWR);
ret = ioctl(fd,0x703,0x48);
if(ret<0){
printf("Could not find i2c device\n");
return ret;}


int fd1, ret1;

char on = '1';

char off = '0';


int limit = 30;

bool isOn = false;

fd1 = open("/sys/class/gpio/gpio26/value",O_RDWR);

if(fd1 == -1){

printf("Fejl ved åbning af GPIO\n");

return -1;
}

while(1){
ret = read(fd,rddata,1);
printf("Temperatur is:  %i\n",rddata[0]);

if(rddata[0] >limit && !isOn){

write(fd1, &on, 1);

} else if(rddata[0] < limit && isOn){

write(fd1, &off, 1);
}
sleep(1);

}

close (fd);
return ret;}
```

## webscript.c:

```
#include <stdbool.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <stdlib.h>
#include <string.h>

int main(){
int fd;
int ret = 0;
char rddata[4], str[16];

fd = open("/dev/i2c-1",O_RDWR);
ret = ioctl(fd,0x703,0x48);
if(ret<0){
printf("Could not find i2c device\n");
return ret;}

char buffer[100];

int fd1, ret1;


while(1){
ret = read(fd,rddata,1);


printf("Temperatur is:  %i\n",rddata[0]);

memset(buffer, 0, sizeof(buffer));  // Clear the buffer
sprintf(buffer, "<html><body><h1>Temperature: %d°C</h1></body></html>", rddata[0]);


fd1 = open("/www/pages/index.html",O_RDWR);

if(fd1 == -1){

printf("Fejl ved åbning af html filen\n");

return -1;
}

write(fd1, buffer, strlen(buffer));

sleep(1);

}

close (fd);
return ret;}
```

# Lektion 3 - Linux Kernel Modules kode:

## hello.c:

```c
#include <linux/init.h>
#include <linux/module.h>
MODULE_LICENSE("Dual BSD/GPL");
static int hello_init(void)
{
 printk(KERN_ALERT "Hello, world\n");
 return 0;
}
static void hello_exit(void)
{
 printk(KERN_ALERT "Goodbye, cruel world\n");
}
module_init(hello_init);
module_exit(hello_exit);
```

# Lektion 4 - Linux char drivers kode:

## readdriver.c:

```c

 #include <linux/gpio.h>
 #include <linux/fs.h>
 #include <linux/cdev.h>
 #include <linux/device.h>
 #include <linux/uaccess.h>
 #include <linux/module.h>

#define buttonGpio 16

const int first_minor = 0;
const int max_devices = 255;
static dev_t devno;
static struct class *mygpio_class;
static struct cdev mygpio_cdev;
struct file_operations mygpio_fops;




 static int mygpio_init(void)
 {
 // Request GPIO
    int gpioRequest;


    gpioRequest = gpio_request(buttonGpio, "SW2 button");


    if(gpioRequest)
    {

        goto but_free;
    }





 // Set GPIO direction (in or out)

    gpio_direction_input(buttonGpio);




 // Alloker Major/Minor
    int err = 0;
    err = alloc_chrdev_region(&devno, first_minor, max_devices,"helloPIO direction (in or out) driver");
    pr_info("Button driver got Major %i\n", MAJOR(devno));

    if(MAJOR(devno) <= 0){
        pr_err( "Failed to allocate dev number\n");
        goto dealoc_dev;
    }


 // Class Create
    mygpio_class = class_create(THIS_MODULE, "mygpio-class");
    if (IS_ERR(mygpio_class))
    {

        pr_err("Failed to create class");
        goto destroy_class;
    };


 // Cdev Init
    cdev_init(&mygpio_cdev, &mygpio_fops);

 // Add Cdev

    err = cdev_add(&mygpio_cdev, devno, 255);
    if (err)
    {
        pr_err("Failed to add cdev");
        goto cdevdel;
    }

    return 0;

    but_free : gpio_free(buttonGpio);
    dealoc_dev : unregister_chrdev_region(devno, max_devices);
    destroy_class : class_destroy(mygpio_class);
    cdevdel :  cdev_del(&mygpio_cdev);

    return err;
}

 static void mygpio_exit(void)
 {
 // Delete Cdev
  cdev_del(&mygpio_cdev);
 // Unregister Device
  unregister_chrdev_region(devno, max_devices);
 // Class Destroy

class_destroy(mygpio_class);
 // Free GPIO

 gpio_free(buttonGpio);

 }


  int mygpio_open(struct inode *inode, struct file *filep)
 {
 int major, minor;
 major = MAJOR(inode->i_rdev);
 minor = MINOR(inode->i_rdev);
 printk("Opening MyGpio Device [major], [minor]: %i, %i\n", major, minor);
 return 0;
 }

 int mygpio_release(struct inode *inode, struct file *filep)
 {
 int minor, major;

 major = MAJOR(inode->i_rdev);
 minor = MINOR(inode->i_rdev);
 printk("Closing/Releasing MyGpio Device [major], [minor]: %i, %i\n", major, minor);

 return 0;
 }





ssize_t mygpio_read(struct file *filep, char __user *ubuf,
size_t count, loff_t *f_pos)
{
char kbuf[12];
int len, value;

value = gpio_get_value(buttonGpio);
len = count < 12 ? count : 12; /* Truncate to smallest */

if(copy_from_user(kbuf, ubuf, len) !=0){
return -EFAULT;
}

pr_info("Wrote %i from minor %i\n", value, iminor(filep->f_inode));

len = snprintf(kbuf, sizeof(kbuf), "%d\n", value); /* Create string */
*f_pos += len; /* Update cursor in file */
return len;
}



 struct file_operations mygpio_fops =
{
   .open = mygpio_open,
   .release = mygpio_release,
   .read = mygpio_read,

};


 module_init(mygpio_init);
 module_exit(mygpio_exit);
 MODULE_LICENSE("GPL");
 MODULE_AUTHOR("Khizer KHan");
 MODULE_DESCRIPTION("");
```

## writedriver.c:

```c

 #include <linux/gpio.h>
 #include <linux/fs.h>
 #include <linux/cdev.h>
 #include <linux/device.h>
 #include <linux/uaccess.h>
 #include <linux/module.h>

#define ledGPio 21

const int first_minor = 0;
const int max_devices = 255;
static dev_t devno;
static struct class *mygpio_class;
static struct cdev mygpio_cdev;
struct file_operations mygpio_fops;

 static int mygpio_init(void)
 {
 // Request GPIO
    int gpioRequest;


    gpioRequest = gpio_request(ledGPio, "LED 3");


    if(gpioRequest)
    {

        goto but_free;
    }





 // Set GPIO direction (in or out)

    gpio_direction_output(ledGPio, 1);




 // Alloker Major/Minor
    int err = 0;
    err = alloc_chrdev_region(&devno, first_minor, max_devices,"helloPIO direction (in or out) driver");
    pr_info("LED driver got Major %i\n", MAJOR(devno));

    if(MAJOR(devno) <= 0){
        pr_err( "Failed to allocate dev number\n");
        goto dealoc_dev;
    }


 // Class Create
    mygpio_class = class_create(THIS_MODULE, "mygpio-class");
    if (IS_ERR(mygpio_class))
    {

        pr_err("Failed to create class");
        goto destroy_class;
    };


 // Cdev Init
    cdev_init(&mygpio_cdev, &mygpio_fops);

 // Add Cdev

    err = cdev_add(&mygpio_cdev, devno, 255);
    if (err)
    {
        pr_err("Failed to add cdev");
        goto cdevdel;
    }

    return 0;

    but_free : gpio_free(ledGPio);
    dealoc_dev : unregister_chrdev_region(devno, max_devices);
    destroy_class : class_destroy(mygpio_class);
    cdevdel :  cdev_del(&mygpio_cdev);

    return err;
}

 static void mygpio_exit(void)
 {
 // Delete Cdev
  cdev_del(&mygpio_cdev);
 // Unregister Device
  unregister_chrdev_region(devno, max_devices);
 // Class Destroy

class_destroy(mygpio_class);
 // Free GPIO

 gpio_free(ledGPio);

 }


ssize_t mygpio_read(struct file *filep, char __user *ubuf,
size_t count, loff_t *f_pos)
{
char kbuf[12];
int len, value;

value = gpio_get_value(ledGPio);
len = count < 12 ? count : 12; /* Truncate to smallest */

if(copy_from_user(kbuf, ubuf, len) !=0){
return -EFAULT;
}

pr_info("Wrote %i from minor %i\n", value, iminor(filep->f_inode));

len = snprintf(kbuf, sizeof(kbuf), "%d\n", value); /* Create string */
*f_pos += len; /* Update cursor in file */
return len;
}

 ssize_t mygpio_write(struct file *filep, const char __user *ubuf,
            size_t count, loff_t *f_pos)
 {

int len, value;
char kbuf[12];
len = count < 12 ? count : 12; /* Truncate to smaller buffer size */
if(copy_from_user(kbuf, ubuf, len) != 0)
return -EFAULT;

sscanf(kbuf, "%d", &value);
gpio_set_value(ledGPio, value);

if (kstrtoint(kbuf, 0, &value)) /* Convert sting to int */
pr_info("Wrote %i from minor %i\n", value, iminor(filep->f_inode));
*f_pos += len; /* Update cursor in file */
return len;

 }

  int mygpio_open(struct inode *inode, struct file *filep)
 {
 int major, minor;
 major = MAJOR(inode->i_rdev);
 minor = MINOR(inode->i_rdev);
 printk("Opening MyGpio Device [major], [minor]: %i, %i\n", major, minor);
 return 0;
 }

 int mygpio_release(struct inode *inode, struct file *filep)
 {
 int minor, major;

 major = MAJOR(inode->i_rdev);
 minor = MINOR(inode->i_rdev);
 printk("Closing/Releasing MyGpio Device [major], [minor]: %i, %i\n", major, minor);

 return 0;
 }




 struct file_operations mygpio_fops =
{
   .open = mygpio_open,
   .release = mygpio_release,
   .read = mygpio_read,
   .write = mygpio_write,

};


 module_init(mygpio_init);
 module_exit(mygpio_exit);
 MODULE_LICENSE("GPL");
 MODULE_AUTHOR("Khizer KHan");
 MODULE_DESCRIPTION("Muhantezi");

```

# Lektion 5 Interrupts kode:

## isr_driver.c:

```c

#include <linux/gpio.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/module.h>
#include <linux/interrupt.h>
#include <linux/wait.h>
#include <linux/sched.h>

static struct class *mygpio_class;
static dev_t devno;
static struct cdev mygpio_cdev;
struct file_operations mygpio_fops;
const int first_minor = 0;
const int max_devices = 255;
int GPIO_SW1 = 12;
static DECLARE_WAIT_QUEUE_HEAD(queue);
static int flag = 0;

static irqreturn_t mygpio_isr(int irq, void *devid)
{
   unsigned int IRQ_SW1 = gpio_to_irq(GPIO_SW1);
   flag = 1;
   wake_up_interruptible (&queue);

   printk("IRQ event at irq line: %i\n", IRQ_SW1);
   return IRQ_HANDLED;
}

static int mygpio_init(void)
{
   // Request GPIO
   gpio_request(GPIO_SW1, "SW1");
   // Set GPIO direction (in or out)
   gpio_direction_input(GPIO_SW1);
   // Alloker Major/Minor
   int err = 0;

   err = alloc_chrdev_region(&devno, first_minor, max_devices, "isr_driver");
   if (err < 0)
   {
      pr_err("Failed to register chardev\n");
      goto fail_A;
   }

   // Class Create
   mygpio_class = class_create(THIS_MODULE, "mygpio-class");
   if (IS_ERR(mygpio_class))
   {
      pr_err("Failed to create class");
      goto fail_B;
   }

   // Cdev Init
   cdev_init(&mygpio_cdev, &mygpio_fops);
   err = cdev_add(&mygpio_cdev, devno, 255);
   if (err)
   {
      pr_err("Failed to add cdev");
      goto fail_C;
   }

   int IRQ_SW1 = gpio_to_irq(GPIO_SW1);
   err = request_irq(IRQ_SW1, mygpio_isr, (IRQF_TRIGGER_FALLING | IRQF_TRIGGER_RISING), "isr_driver", NULL);
   if (err)
   {
      pr_err("Failed to Request");
      goto fail_D;
   }

   printk("ISR linje: [IRQ_SW1] : %i\n,", IRQ_SW1);
   return 0;
fail_D:
   cdev_del(&mygpio_cdev);
fail_C:
   class_destroy(mygpio_class);
fail_B:
   unregister_chrdev_region(devno, max_devices);
fail_A:
   return err;
}

static void mygpio_exit(void)
{
   unsigned int IRQ_SW1 = gpio_to_irq(GPIO_SW1);
   free_irq(IRQ_SW1, NULL);

   gpio_free(GPIO_SW1);

   // Delete Cdev
   cdev_del(&mygpio_cdev);

   // Unregister Device
   unregister_chrdev_region(devno, max_devices);

   // Class Destroy
   class_destroy(mygpio_class);
}

int mygpio_open(struct inode *inode, struct file *filep)
{
   int major, minor;
   major = MAJOR(inode->i_rdev);
   minor = MINOR(inode->i_rdev);
   printk("Opening MyGpio Device [major], [minor]: %i, %i\n", major, minor);
   return 0;
}

int mygpio_release(struct inode *inode, struct file *filep)
{
   int minor, major;

   major = MAJOR(inode->i_rdev);
   minor = MINOR(inode->i_rdev);
   printk("Closing/Releasing MyGpio Device [major], [minor]: %i, %i\n", major, minor);

   return 0;
}

ssize_t mygpio_read(struct file *filep, char __user *buf, size_t count, loff_t *f_pos)
{
   char kbuf[12];
   int len, value;
   value = gpio_get_value(GPIO_SW1);
   len = count < 12 ? count : 12;
   len = snprintf(kbuf, len, "%i", value);

   wait_event_interruptible(queue, flag != 0);
   flag = 0;

   if (copy_to_user(buf, kbuf, ++len))
   {
      return -EFAULT;
   }

   *f_pos += len;
   return len;
}

struct file_operations mygpio_fops = {
    .owner = THIS_MODULE,
    .open = mygpio_open,
    .release = mygpio_release,
    .read = mygpio_read};

module_init(mygpio_init);
module_exit(mygpio_exit);
MODULE_AUTHOR("Andreas K.");
MODULE_LICENSE("GPL");


```

## isr_driver.mod.c:

```c
#include <linux/build-salt.h>
#include <linux/module.h>
#include <linux/vermagic.h>
#include <linux/compiler.h>

BUILD_SALT;

MODULE_INFO(vermagic, VERMAGIC_STRING);
MODULE_INFO(name, KBUILD_MODNAME);

__visible struct module __this_module
__section(.gnu.linkonce.this_module) = {
	.name = KBUILD_MODNAME,
	.init = init_module,
#ifdef CONFIG_MODULE_UNLOAD
	.exit = cleanup_module,
#endif
	.arch = MODULE_ARCH_INIT,
};

#ifdef CONFIG_RETPOLINE
MODULE_INFO(retpoline, "Y");
#endif

static const struct modversion_info ____versions[]
__used __section(__versions) = {
};

MODULE_INFO(depends, "");


MODULE_INFO(srcversion, "E683E8908F0326BA1158759");
```

# Lektion 6 Linux Device Model & Device Tree kode:

## plat_drv.c

```c
#include <linux/gpio.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/of_gpio.h>
#include <linux/of_address.h>
#include <linux/of_device.h>
#include <linux/of_irq.h>
#include <linux/kernel.h>

#define ledGPio 21

const int first_minor = 0;
const int max_devices = 255;
static dev_t devno;
static struct class *mygpio_class;
static struct cdev mygpio_cdev;
struct device *gpio_device;
static struct platform_driver my_p_driver;  // Forward declaration
static const u8 gpio = 21;
static const u8 minor = 0;
static int gpios_len = 2;


struct gpio_dev {
  int no;   // GPIO number
  int dir; // 0: in, 1: out
};

int gpio_devs_cnt = 0;
static struct gpio_dev gpio_devs[255];

struct file_operations mygpio_fops;

static int mygpio_init(void)
{
    int gpioRequest, err;

    // Request GPIO
    gpioRequest = gpio_request(ledGPio, "LED 3");
    if (gpioRequest) {
        goto free_gpio;
    }

    // Set GPIO direction (output)
    gpio_direction_output(ledGPio, 1);

    // Allocate Major/Minor
    err = alloc_chrdev_region(&devno, first_minor, max_devices, "helloPIO");
    if (err < 0) {
        pr_err("Failed to allocate device number\n");
        goto free_gpio;
    }
    pr_info("LED driver got Major %i\n", MAJOR(devno));

    // Create class
    mygpio_class = class_create(THIS_MODULE, "mygpio-class");
    if (IS_ERR(mygpio_class)) {
        pr_err("Failed to create class\n");
        err = PTR_ERR(mygpio_class);
        goto unregister_dev;
    }

    // Initialize and add cdev
    cdev_init(&mygpio_cdev, &mygpio_fops);
    err = cdev_add(&mygpio_cdev, devno, max_devices);
    if (err) {
        pr_err("Failed to add cdev\n");
        goto destroy_class;
    }

    // Register platform driver
    err = platform_driver_register(&my_p_driver);
    if (err) {
        pr_err("Failed to register platform driver\n");
        goto delete_cdev;
    }

    return 0;

free_gpio:
    gpio_free(ledGPio);
unregister_dev:
    unregister_chrdev_region(devno, max_devices);
destroy_class:
    class_destroy(mygpio_class);
delete_cdev:
    cdev_del(&mygpio_cdev);
    return err;
}

static void mygpio_exit(void)
{
    platform_driver_unregister(&my_p_driver);
    cdev_del(&mygpio_cdev);
    unregister_chrdev_region(devno, max_devices);
    class_destroy(mygpio_class);

    for (unsigned int i = 0; i < gpios_len; i++) {
        gpio_free(gpio_devs[i].no);
        printk("Freed GPIO %d in exit\n", gpio_devs[i].no);
    }
}

ssize_t mygpio_read(struct file *filep, char __user *ubuf, size_t count, loff_t *f_pos)
{
    char kbuf[12];
    int len, value, minor;

    minor = iminor(filep->f_inode);

    value = gpio_get_value(gpio_devs[minor].no);

    pr_info("Read value %d from GPIO %d (minor %d)\n", value, gpio_devs[minor].no, minor);


    len = snprintf(kbuf, sizeof(kbuf), "%d\n", value);

    if (copy_to_user(ubuf, kbuf, len)) {
        return -EFAULT;
    }
    *f_pos += len;
    return len;
}

ssize_t mygpio_write(struct file *filep, const char __user *ubuf, size_t count, loff_t *f_pos)
{
    char kbuf[12];
    int value, len, minor;

    minor = iminor(filep->f_inode);

    len = min(count, sizeof(kbuf) - 1);
    if (copy_from_user(kbuf, ubuf, len)) {
        return -EFAULT;
    }
    kbuf[len] = '\0';

    if (kstrtoint(kbuf, 10, &value) < 0) {
        pr_err("Invalid value\n");
        return -EINVAL;
    }

    gpio_set_value(gpio_devs[minor].no, value);
    pr_info("Wrote %d to GPIO %d\n", value, gpio_devs[minor].no);
    *f_pos += len;
    return len;
}

int mygpio_open(struct inode *inode, struct file *filep)
{
    int major = MAJOR(inode->i_rdev);
    int minor = MINOR(inode->i_rdev);
    pr_info("Opening MyGpio Device [major], [minor]: %d, %d\n", major, minor);
    return 0;
}

int mygpio_release(struct inode *inode, struct file *filep)
{
    int major = MAJOR(inode->i_rdev);
    int minor = MINOR(inode->i_rdev);
    pr_info("Closing MyGpio Device [major], [minor]: %d, %d\n", major, minor);
    return 0;
}

static int my_p_drv_probe(struct platform_device *pdev)
{
  int err = 0;
  struct device *dev = &pdev->dev; // Device ptr derived from current platform_device
  struct device_node *np = dev->of_node; // Device tree node ptr
  enum of_gpio_flags flag;

           printk(KERN_INFO "plat_drv probe function called\n");

  int gpios_in_dt = of_gpio_count(np);

       printk(KERN_INFO "Number of GPIOs in (method caller: probe) device tree: %d\n", gpios_in_dt);


  if (gpios_in_dt < 0)
  {
        dev_err(dev, "Failed to retrieve GPIO count\n");
        return gpios_in_dt;
  }

  for (int i = 0; i < gpios_in_dt; i++)
  {
        gpio_devs[i].no = of_get_gpio(np, i);
        if (gpio_devs[i].no < 0) {
            dev_err(dev, "Failed to get GPIO %d\n", i);
            return gpio_devs[i].no;
        }


    gpio_devs[i].dir = of_get_gpio_flags(np, i, &flag);
    gpio_devs_cnt++;


  }
    for(unsigned int i = 0; i < gpios_in_dt; i++)
    {
    printk("Hello from probe: %s\n", pdev->name);
/* Request resources */
    gpio_request(gpio_devs[i].no, "my_p_dev_gpio");

    if (gpio_devs[i].dir == 0) {
        gpio_direction_input(gpio_devs[i].no);
    } else {
        gpio_direction_output(gpio_devs[i].no, 0); // Set initial output value to 0
    }

/* Dynamically add device */
    gpio_device = device_create(mygpio_class, NULL, MKDEV(MAJOR(devno), gpio_devs[i].dir), NULL, "mygpio%d", gpio_devs[i].no);

      printk(KERN_ALERT "Using GPIO[%i], flag:%i on major:%i, minor:%i\n",
             gpio_devs[gpio_devs_cnt].no, gpio_devs[gpio_devs_cnt].dir,
             MKDEV(MAJOR(devno), gpio_devs[gpio_devs_cnt].no), gpio_devs_cnt);
    }

    return 0;
}

static int my_p_drv_remove(struct platform_device *pdev)
{
     struct device *dev = &pdev->dev; // Device ptr derived from current platform_device
     struct device_node *np = dev->of_node;

         printk(KERN_INFO "plat_drv remove function called\n");

     int gpios_in_dt = of_gpio_count(np);



    for(unsigned int i = 0; i < gpios_in_dt; i++)
    {

    printk("Removing platform driver: %s\n", pdev->name);
    /* Remove device created in probe, this must be
    * done for all devices created in probe */
    device_destroy(mygpio_class,
    MKDEV(MAJOR(devno), gpio_devs[i].dir));
    gpio_free(gpio_devs[i].no);
    printk("Freed GPIO %d for device %d\n", gpio_devs[i].no, i);

    }

    return 0;
}

static const struct of_device_id my_p_drv_of_match[] = {
    { .compatible = "au-ece,plat_drv" },
    {},
};

MODULE_DEVICE_TABLE(of, my_p_drv_of_match);

static struct platform_driver my_p_driver = {
    .driver = {
        .name = "my_platform_driver",
        .owner = THIS_MODULE,
        .of_match_table = my_p_drv_of_match,
    },
    .probe = my_p_drv_probe,
    .remove = my_p_drv_remove,
};

struct file_operations mygpio_fops = {
    .open = mygpio_open,
    .release = mygpio_release,
    .read = mygpio_read,
    .write = mygpio_write,
};

module_init(mygpio_init);
module_exit(mygpio_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Khizer Khan");
MODULE_DESCRIPTION("GPIO Platform Driver");

```

## plat_drv-overlays.dts :

```c
// Definitions for plat_drv module
// Adapted from w1-gpio-pullup-overlay.dts
/dts-v1/;
/plugin/;

/ {
  compatible = "brcm,bcm2835", "brcm,bcm2836", "brcm,bcm2708", "brcm,bcm2709";

  fragment@0 {
    /* Add device to base */
    target-path = "/";
    __overlay__ {
      /* instance:type */
      plat_drv: plat_drv@0 {
        /* Label to match in driver */
        compatible = "au-ece,plat_drv";
        gpios = <&gpio 12 0>, <&gpio 21 1>;
        status = "okay";
        mydevt-custom = <0x12345678>;
      };
    };
  };
};
```

# Lektion 7: Linux Device Model & Bus Interface

## spi_drv.c :

```c

#include <linux/cdev.h>      // cdev_add, cdev_init
#include <linux/module.h>    // module_init, GPL
#include <linux/spi/spi.h>   // spi_sync,
#include <linux/uaccess.h>   // copy_to_user

#define MAXLEN       32
#define MODULE_DEBUG 1   // Enable/Disable Debug messages

/* Char Driver Globals */
static struct spi_driver spi_drv_spi_driver;
struct file_operations spi_drv_fops;
static struct class *spi_drv_class;
static dev_t devno;
static struct cdev spi_drv_cdev;

/* Definition of SPI devices */
struct Myspi
{
    struct spi_device *spi;   // Pointer to SPI device
    int channel;              // channel, ex. adc ch 0
};
/* Array of SPI devices */
/* Minor used to index array */
struct Myspi spi_devs[ 4 ];
const int spi_devs_len = 4;    // Max nbr of devices
static int spi_devs_cnt = 0;   // Nbr devices present

/* Macro to handle Errors */
#define ERRGOTO( label, ... )  \
    {                          \
        printk( __VA_ARGS__ ); \
        goto label;            \
    }                          \
    while ( 0 )

/**********************************************************
 * CHARACTER DRIVER METHODS
 **********************************************************/

/*
 * Character Driver Module Init Method
 */
static int __init spi_drv_init( void )
{
    int err = 0;

    printk( "spi_drv driver initializing\n" );

    /* Allocate major number and register fops*/
    err = alloc_chrdev_region( &devno, 0, 255, "spi_drv driver" );
    if ( MAJOR( devno ) <= 0 )
        ERRGOTO( err_no_cleanup, "Failed to register chardev\n" );
    printk( KERN_ALERT "Assigned major no: %i\n", MAJOR( devno ) );

    cdev_init( &spi_drv_cdev, &spi_drv_fops );
    err = cdev_add( &spi_drv_cdev, devno, 255 );
    if ( err )
        ERRGOTO( err_cleanup_chrdev, "Failed to create class" );

    /* Polulate sysfs entries */
    spi_drv_class = class_create( THIS_MODULE, "spi_drv_class" );
    if ( IS_ERR( spi_drv_class ) )
        ERRGOTO( err_cleanup_cdev, "Failed to create class" );

    /* Register SPI Driver */
    /* THIS WILL INVOKE PROBE, IF DEVICE IS PRESENT!!! */
    err = spi_register_driver( &spi_drv_spi_driver );
    if ( err )
        ERRGOTO( err_cleanup_class, "Failed SPI Registration\n" );

    /* Success */
    return 0;

    /* Errors during Initialization */
err_cleanup_class:
    class_destroy( spi_drv_class );

err_cleanup_cdev:
    cdev_del( &spi_drv_cdev );

err_cleanup_chrdev:
    unregister_chrdev_region( devno, 255 );

err_no_cleanup:
    return err;
}

/*
 * Character Driver Module Exit Method
 */
static void __exit spi_drv_exit( void )
{
    printk( "spi_drv driver Exit\n" );

    spi_unregister_driver( &spi_drv_spi_driver );
    class_destroy( spi_drv_class );
    cdev_del( &spi_drv_cdev );
    unregister_chrdev_region( devno, 255 );
}

/*
 * Character Driver Write File Operations Method
 */
ssize_t spi_drv_write( struct file *filep,
                       const char __user *ubuf,
                       size_t count,
                       loff_t *f_pos )
{
    int minor, len, value;
    char kbuf[ MAXLEN ];

    minor = iminor( filep->f_inode );

    printk( KERN_ALERT "Writing to spi_drv [Minor] %i \n", minor );

    /* Limit copy length to MAXLEN allocated andCopy from user */
    len = count < MAXLEN ? count : MAXLEN;
    if ( copy_from_user( kbuf, ubuf, len ) )
        return -EFAULT;

    /* Pad null termination to string */
    kbuf[ len ] = '\0';

    if ( MODULE_DEBUG )
        printk( "string from user: %s\n", kbuf );

    /* Convert sting to int */
    sscanf( kbuf, "%i", &value );
    if ( MODULE_DEBUG )
        printk( "value %i\n", value );

    /*
      Do something with value ....
    */

    /* Legacy file ptr f_pos. Used to support
     * random access but in char drv we dont!
     * Move it the length actually  written
     * for compability */
    *f_pos += len;

    /* return length actually written */
    return len;
}

int my_spi_func( struct spi_device *sdev, int channel, u16 *result )
{

    struct spi_transfer t[ 3 ];
    struct spi_message m;

    //Alle bytes i t array bliver sat til 0.
    memset( t, 0, sizeof( t ) );

    //Initialisering af spi_message m
    spi_message_init( &m );

    //Dette sætter messagens pointer attribut til det samme som spi device pointer
    m.spi = sdev;

    //Variabel til at gemme data fra slave i anden transfer
    u8 data1;

    //Variabel til at gemme data fra slave i tredje transfer
    u8 data2;

    //Byte som master skal sende til slave i første transfer
    u8 start = 0b00000001;

    //Denne variabel angiver hvilken kanal der skal modtages data fra
    u8 mode = 0b10100000 | ( channel << 6 );

//Første transfer
    t[ 0 ].tx_buf = &start;
    t[ 0 ].rx_buf = NULL;
    t[ 0 ].len = 1;
    spi_message_add_tail( &t[ 0 ], &m );

//Anden transfer
    t[ 1 ].tx_buf = &mode;
    t[ 1 ].rx_buf = &data1;
    t[ 1 ].len = 1;
    spi_message_add_tail( &t[ 1 ], &m );

//Tredje transfer
    t[ 2 ].tx_buf = 0;
    t[ 2 ].rx_buf = &data2;
    t[ 2 ].len = 1;
    spi_message_add_tail( &t[ 2 ], &m );

    //Her udføres spi transaktionen synkront.
    spi_sync( m.spi, &m );

    //For at eliminere de 4 bit mest til venstre så & vi med 0x0F(0000 1111)
    //Derefter shifter til venstre 8 gange. Derefter OR vi med data2.
    *result = ( ( data1 & 0x0F ) << 8 ) | data2;

    //Udfra formlen i datablad beregnes Vin
    *result = ( ( *result ) * 3300 ) / 4096;
    return 0;
}

/*
 * Character Driver Read File Operations Method
 */
ssize_t spi_drv_read( struct file *filep,
                      char __user *ubuf,
                      size_t count,
                      loff_t *f_pos )
{
    int minor, len;
    char resultBuf[ MAXLEN ];
    s16 result = 1234;

    minor = iminor( filep->f_inode );

    /*
      Provide a result to write to user space
      spidevs[minor].spi,
    */
    my_spi_func( spi_devs[ minor ].spi, spi_devs[ minor ].channel, &result );

    if ( MODULE_DEBUG )
        printk( KERN_ALERT "%s-%i read: %i\n",
                spi_devs[ minor ].spi->modalias,
                spi_devs[ minor ].channel,
                result );

    /* Convert integer to string limited to "count" size. Returns
     * length excluding NULL termination */
    len = snprintf( resultBuf, count, "%d\n", result );

    /* Append Length of NULL termination */
    len++;

    /* Copy data to user space */
    if ( copy_to_user( ubuf, resultBuf, len ) )
        return -EFAULT;

    /* Move fileptr */
    *f_pos += len;

    return len;
}

/*
 * Character Driver File Operations Structure
 */
struct file_operations spi_drv_fops = {
    .owner = THIS_MODULE,
    .write = spi_drv_write,
    .read = spi_drv_read,
};

/**********************************************************
 * LINUX DEVICE MODEL METHODS (spi)
 **********************************************************/

/*
 * spi_drv Probe
 * Called when a device with the name "spi_drv" is
 * registered.
 */
static int spi_drv_probe( struct spi_device *sdev )
{
    int err = 0;
    struct device *spi_drv_device;

    printk( KERN_DEBUG "New SPI device: %s using chip select: %i\n",
            sdev->modalias,
            sdev->chip_select );

    /* Check we are not creating more
       devices than we have space for */
    if ( spi_devs_cnt > spi_devs_len )
    {
        printk( KERN_ERR "Too many SPI devices for driver\n" );
        return -ENODEV;
    }

    /* Configure bits_per_word, always 8-bit for RPI!!! */
    sdev->bits_per_word = 8;
    spi_setup( sdev );

    /* Create devices, populate sysfs and
       active udev to create devices in /dev */

    for ( int i = 0; i < 2; i++ )
    {
        /* We map spi_devs index to minor number here */
        spi_drv_device = device_create( spi_drv_class,
                                        NULL,
                                        MKDEV( MAJOR( devno ), spi_devs_cnt ),
                                        NULL,
                                        "spi_drv%d",
                                        spi_devs_cnt );
        if ( IS_ERR( spi_drv_device ) )
            printk( KERN_ALERT "FAILED TO CREATE DEVICE\n" );
        else
            printk( KERN_ALERT "Using spi_devs%i on major:%i, minor:%i\n",
                    spi_devs_cnt,
                    MAJOR( devno ),
                    spi_devs_cnt );

        /* Update local array of SPI devices */
        spi_devs[ spi_devs_cnt ].spi = sdev;
        spi_devs[ spi_devs_cnt ].channel = 0x00 + i;   // channel address

        ++spi_devs_cnt;
    }

    return err;
}

/*
 * spi_drv Remove
 * Called when the device is removed
 * Can deallocate data if needed
 */
static int spi_drv_remove( struct spi_device *sdev )
{
    for ( int i = 0; i < 2; i++ )
    {
        int its_minor = 0 + i;

        printk( KERN_ALERT "Removing spi device\n" );

        /* Destroy devices created in probe() */
        device_destroy( spi_drv_class, MKDEV( MAJOR( devno ), its_minor ) );
    }

    return 0;
}

/*
 * spi Driver Struct
 * Holds function pointers to probe/release
 * methods and the name under which it is registered
 */
static const struct of_device_id of_spi_drv_spi_device_match[] = {
    {
        .compatible = "ase, spi_drv",
    },
    {},
};

static struct spi_driver spi_drv_spi_driver = {
    .probe = spi_drv_probe,
    .remove = spi_drv_remove,
    .driver =
        {
            .name = "spi_drv",
            .bus = &spi_bus_type,
            .of_match_table = of_spi_drv_spi_device_match,
            .owner = THIS_MODULE,
        },
};

/**********************************************************
 * GENERIC LINUX DEVICE DRIVER STUFF
 **********************************************************/

/*
 * Assignment of module init/exit methods
 */
module_init( spi_drv_init );
module_exit( spi_drv_exit );

/*
 * Assignment of author and license
 */
MODULE_AUTHOR( "Peter Hoegh Mikkelsen <phm@ase.au.dk>" );
MODULE_LICENSE( "GPL" );

```

## spi_drv-overlay.dts :

```c
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2835", "brcm,bcm2708";
    /* disable spi-dev for spi0.0 */
    fragment@0 {
        target = <&spi0>; // SPI Bus 0
        __overlay__ {
            status = "okay";
            spidev@0{     // SPI Chip Select 0
                status = "disabled";
            };
        };
    };

    fragment@1 {
        target = <&spi0>; // SPI Bus 0
        __overlay__ {
            /* needed to avoid dtc warning */
            #address-cells = <1>;
            #size-cells = <0>;
            spi_drv:spi_drv@0 {
                compatible = "ase, spi_drv";
                reg = <0>; // SPI Chip Select 5
                spi-cpha=<0>;  /* CPHA flag, remove comment to set flag */
                spi-cpol=<0>;  /* CPOL flag, remove comment to set flag  */
                spi-max-frequency = <1000000>; /* SPI frequency in Hz */
            };
        };
    };
};
```

# Lektion 08: Device Model, Attributes, Timers and Deferred Work kode:

## opgave2.c :

```c
#include <linux/gpio.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/of_gpio.h>
#include <linux/of_address.h>
#include <linux/of_device.h>
#include <linux/of_irq.h>
#include <linux/kernel.h>
#include <linux/timer.h>
#include <linux/sched.h>

#define ledGPio 21
#define TOGGLE_OFF 0
#define TOGGLE_ON 1

const int first_minor = 0;
const int max_devices = 255;
static dev_t devno;
static struct class *mygpio_class;
static struct cdev mygpio_cdev;
struct device *gpio_device;
static struct platform_driver my_p_driver;  // Forward declaration
static const u8 gpio = 21;
static const u8 minor = 0;
static int gpios_len = 2;

/* Global Variable */
static int toggle_state;

static int gpio_value = 0;

struct gpio_dev {
  int no;   // GPIO number
  int dir; // 0: in, 1: out
  int toggle_state;
  int toggle_delay;
  struct timer_list timer;
};

int gpio_devs_cnt = 0;
static struct gpio_dev gpio_devs[255];

struct file_operations mygpio_fops;
static void gpio_timer_callback(struct timer_list *t)
{
    struct gpio_dev *gpio = from_timer(gpio, t, timer);

    gpio->toggle_state = gpio->toggle_state ^1;
    gpio_set_value(gpio->no, gpio->toggle_state);

    mod_timer(&gpio->timer, jiffies + msecs_to_jiffies(gpio->toggle_delay));
}


static ssize_t gpio_toggle_state_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    struct gpio_dev *gpio = dev_get_drvdata(dev);
    return sprintf(buf, "%d\n", gpio->toggle_state);
}
static ssize_t gpio_toggle_state_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t size)
{
    struct gpio_dev *gpio = dev_get_drvdata(dev);
    int value;

    kstrtoint(buf, 0, &value);

    gpio->toggle_state = value;


    return size;
}

static ssize_t gpio_toggle_delay_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    struct gpio_dev *gpio = dev_get_drvdata(dev);
    return sprintf(buf, "%d\n", gpio->toggle_delay);
}

static ssize_t gpio_toggle_delay_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t size)
{
    struct gpio_dev *gpio = dev_get_drvdata(dev);
    int value;

    int err = kstrtoint(buf, 0, &value);
    if (err<0){
        return err;
    }
    gpio->toggle_delay = value;


    return size;
}

DEVICE_ATTR_RW(gpio_toggle_state);
DEVICE_ATTR_RW(gpio_toggle_delay);

static struct attribute *gpio_attrs[] = {
    &dev_attr_gpio_toggle_state.attr,
    &dev_attr_gpio_toggle_delay.attr,
    NULL,
};
ATTRIBUTE_GROUPS(gpio);


static ssize_t toggle_state_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    /* Retrieve drvdata ptr (set in device_create) */
    struct gpio_dev *d = dev_get_drvdata(dev);
    int value = 1;
    int len = sprintf(buf, "%d\n", toggle_state);
    return len;
}

static ssize_t toggle_state_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t size)
{
    struct gpio_dev *d = dev_get_drvdata(dev);
    int value;
    int err = kstrtoint(buf, 10, &value); // Parse input string
    if (err < 0) {
        printk(KERN_ALERT "Unable to parse string\n");
        return err;
    }
    d->toggle_delay=1000;
    d->toggle_state=value;
    printk(KERN_INFO "Updated toggle_state to %d\n", value);
        if(value==1){
             timer_setup(&d->timer, gpio_timer_callback, 0);
            mod_timer(&d->timer, jiffies + msecs_to_jiffies(d->toggle_delay));
        }else if(value==0){
            del_timer_sync(&d->timer);
        }
    return size;
}


// DEVICE_ATTR_RW(toggle_state);
// /* Attribute array of all led attributes */
// static struct attribute *toggle_attrs[] = {
// &dev_attr_toggle_state.attr,
// // More attributes could go in here,
// NULL,
// };
// ATTRIBUTE_GROUPS(toggle);


static int mygpio_init(void)
{
    int gpioRequest, err;

    // Request GPIO
    gpioRequest = gpio_request(ledGPio, "LED 3");
    if (gpioRequest) {
        goto free_gpio;
    }

    // Set GPIO direction (output)
    gpio_direction_output(ledGPio, 1);

    // Allocate Major/Minor
    err = alloc_chrdev_region(&devno, first_minor, max_devices, "helloPIO");
    if (err < 0) {
        pr_err("Failed to allocate device number\n");
        goto free_gpio;
    }
    pr_info("LED driver got Major %i\n", MAJOR(devno));

    // Create class
    mygpio_class = class_create(THIS_MODULE, "mygpio-class");
    if (IS_ERR(mygpio_class)) {
        pr_err("Failed to create class\n");
        err = PTR_ERR(mygpio_class);
        goto unregister_dev;
    }

  //  mygpio_class->dev_groups = toggle_groups;
    mygpio_class->dev_groups = gpio_groups;



    // Initialize and add cdev
    cdev_init(&mygpio_cdev, &mygpio_fops);
    err = cdev_add(&mygpio_cdev, devno, max_devices);
    if (err) {
        pr_err("Failed to add cdev\n");
        goto destroy_class;
    }

    // Register platform driver
    err = platform_driver_register(&my_p_driver);
    if (err) {
        pr_err("Failed to register platform driver\n");
        goto delete_cdev;
    }

    return 0;

free_gpio:
    gpio_free(ledGPio);
unregister_dev:
    unregister_chrdev_region(devno, max_devices);
destroy_class:
    class_destroy(mygpio_class);
delete_cdev:
    cdev_del(&mygpio_cdev);
    return err;
}

static void mygpio_exit(void)
{
    platform_driver_unregister(&my_p_driver);
    cdev_del(&mygpio_cdev);
    unregister_chrdev_region(devno, max_devices);
    class_destroy(mygpio_class);

    for (unsigned int i = 0; i < gpios_len; i++) {
        gpio_free(gpio_devs[i].no);
        printk("Freed GPIO %d in exit\n", gpio_devs[i].no);
    }
}

ssize_t mygpio_read(struct file *filep, char __user *ubuf, size_t count, loff_t *f_pos)
{
    char kbuf[12];
    int len, value, minor;

    minor = iminor(filep->f_inode);

    value = gpio_get_value(gpio_devs[minor].no);

    pr_info("Read value %d from GPIO %d (minor %d)\n", value, gpio_devs[minor].no, minor);


    len = snprintf(kbuf, sizeof(kbuf), "%d\n", value);

    if (copy_to_user(ubuf, kbuf, len)) {
        return -EFAULT;
    }
    *f_pos += len;
    return len;
}

ssize_t mygpio_write(struct file *filep, const char __user *ubuf, size_t count, loff_t *f_pos)
{
    char kbuf[12];
    int value, len, minor;

    minor = iminor(filep->f_inode);

    len = min(count, sizeof(kbuf) - 1);
    if (copy_from_user(kbuf, ubuf, len)) {
        return -EFAULT;
    }
    kbuf[len] = '\0';

    if (kstrtoint(kbuf, 10, &value) < 0) {
        pr_err("Invalid value\n");
        return -EINVAL;
    }

    gpio_set_value(gpio_devs[minor].no, value);
    pr_info("Wrote %d to GPIO %d\n", value, gpio_devs[minor].no);
    *f_pos += len;
    return len;
}

int mygpio_open(struct inode *inode, struct file *filep)
{
    int major = MAJOR(inode->i_rdev);
    int minor = MINOR(inode->i_rdev);
    pr_info("Opening MyGpio Device [major], [minor]: %d, %d\n", major, minor);
    return 0;
}

int mygpio_release(struct inode *inode, struct file *filep)
{
    int major = MAJOR(inode->i_rdev);
    int minor = MINOR(inode->i_rdev);
    pr_info("Closing MyGpio Device [major], [minor]: %d, %d\n", major, minor);
    return 0;
}

static int my_p_drv_probe(struct platform_device *pdev)
{
  int err = 0;
  struct device *dev = &pdev->dev; // Device ptr derived from current platform_device
  struct device_node *np = dev->of_node; // Device tree node ptr
  enum of_gpio_flags flag;

           printk(KERN_INFO "plat_drv probe function called\n");

  int gpios_in_dt = of_gpio_count(np);

       printk(KERN_INFO "Number of GPIOs in (method caller: probe) device tree: %d\n", gpios_in_dt);


  if (gpios_in_dt < 0)
  {
        dev_err(dev, "Failed to retrieve GPIO count\n");
        return gpios_in_dt;
  }

  for (int i = 0; i < gpios_in_dt; i++)
  {
        gpio_devs[i].no = of_get_gpio(np, i);
        if (gpio_devs[i].no < 0) {
            dev_err(dev, "Failed to get GPIO %d\n", i);
            return gpio_devs[i].no;
        }


    gpio_devs[i].dir = of_get_gpio_flags(np, i, &flag);
    gpio_devs_cnt++;


  }
    for(unsigned int i = 0; i < gpios_in_dt; i++)
    {
    printk("Hello from probe: %s\n", pdev->name);
/* Request resources */
    gpio_request(gpio_devs[i].no, "my_p_dev_gpio");

    if (gpio_devs[i].dir == 0) {
        gpio_direction_input(gpio_devs[i].no);
    } else {
        gpio_direction_output(gpio_devs[i].no, 0); // Set initial output value to 0
    }

/* Dynamically add device */
    gpio_device = device_create(mygpio_class, NULL, MKDEV(MAJOR(devno), gpio_devs[i].dir), &gpio_devs[i], "mygpio%d", gpio_devs[i].dir);

      printk(KERN_ALERT "Using GPIO[%i], flag:%i on major:%i, minor:%i\n",
             gpio_devs[gpio_devs_cnt].no, gpio_devs[gpio_devs_cnt].dir,
             MKDEV(MAJOR(devno), gpio_devs[gpio_devs_cnt].no), gpio_devs_cnt);
    }

    return 0;
}

static int my_p_drv_remove(struct platform_device *pdev)
{
     struct device *dev = &pdev->dev; // Device ptr derived from current platform_device
     struct device_node *np = dev->of_node;

         printk(KERN_INFO "plat_drv remove function called\n");

     int gpios_in_dt = of_gpio_count(np);



    for(unsigned int i = 0; i < gpios_in_dt; i++)
    {

    printk("Removing platform driver: %s\n", pdev->name);
    /* Remove device created in probe, this must be
    * done for all devices created in probe */
    device_destroy(mygpio_class,
    MKDEV(MAJOR(devno), gpio_devs[i].dir));
    gpio_free(gpio_devs[i].no);
    del_timer_sync(&gpio_devs[i].timer);
    printk("Freed GPIO %d for device %d\n", gpio_devs[i].no, i);

    }

    return 0;
}

static const struct of_device_id my_p_drv_of_match[] = {
    { .compatible = "au-ece,plat_drv" },
    {},
};

MODULE_DEVICE_TABLE(of, my_p_drv_of_match);

static struct platform_driver my_p_driver = {
    .driver = {
        .name = "my_platform_driver",
        .owner = THIS_MODULE,
        .of_match_table = my_p_drv_of_match,
    },
    .probe = my_p_drv_probe,
    .remove = my_p_drv_remove,
};

struct file_operations mygpio_fops = {
    .open = mygpio_open,
    .release = mygpio_release,
    .read = mygpio_read,
    .write = mygpio_write,
};

module_init(mygpio_init);
module_exit(mygpio_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Khizer Khan");
MODULE_DESCRIPTION("GPIO Platform Driver");
```
