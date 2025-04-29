# k230_canmv
*这篇文章是我在学习k230的过程中，对canmv的一些学习笔记。*
## 1.Sensor
### 简介
    * 在canmv中sensor指代的是摄像头。
        * 摄像头的种类有：  
          * 1.OV2640
          * 2.OV5640
          * 3.OV7725
    * k230可以同时使用多个摄像头。分别为id0 1 2
  
### 调用流程
    * 1.导入库
    * 2.reset 摄像头 sensor.reset()
    * 3.设置摄像头参数
    * 4.绑定显示(可选 绑定的作用是不需要后面再手动读取)
      * 这里注意：貌似是不可以选择绑定到IDE上的图像像素格式的 
      * 绑定到IDE上的图像像素格式只能是YUV420SP
      * 结合Display使用
    * display.init(),media.init()
    * 5.开始摄像头  
      * '''sensor.run()'''
  * 代码如下：
  ```python
  # Camera 示例
import time
import os
import sys

from media.sensor import *
from media.display import *
from media.media import *

sensor = None

try:
    print("camera_test")

    # 根据默认配置构建 Sensor 对象
    sensor = Sensor()
    # 复位 sensor
    sensor.reset()

    # 设置通道 0 分辨率为 1920x1080
    sensor.set_framesize(Sensor.FHD)
    # 设置通道 0 格式为 YUV420SP
    sensor.set_pixformat(Sensor.YUV420SP)
    # 绑定通道 0 到显示 VIDEO1 层
    bind_info = sensor.bind_info()
    Display.bind_layer(**bind_info, layer=Display.LAYER_VIDEO1)

    # 设置通道 1 分辨率和格式
    sensor.set_framesize(width=640, height=480, chn=CAM_CHN_ID_1)
    sensor.set_pixformat(Sensor.RGB888, chn=CAM_CHN_ID_1)

    # 设置通道 2 分辨率和格式
    sensor.set_framesize(width=640, height=480, chn=CAM_CHN_ID_2)
    sensor.set_pixformat(Sensor.RGB565, chn=CAM_CHN_ID_2)

    # 初始化 HDMI 和 IDE 输出显示，若屏幕无法点亮，请参考 API 文档中的 K230_CanMV_Display 模块 API 手册进行配置
    Display.init(Display.LT9611, to_ide=True, osd_num=2)#osd_num=2 是指显示的图层数
    # 初始化媒体管理器
    MediaManager.init()
    # 启动 sensor
    sensor.run()

    while True:
        os.exitpoint()

        img = sensor.snapshot(chn=CAM_CHN_ID_1)
        Display.show_image(img, alpha=128)#alpha=128 是指透明度,0-255

        img = sensor.snapshot(chn=CAM_CHN_ID_2)
        Display.show_image(img, x=1920 - 640, layer=Display.LAYER_OSD1)

except KeyboardInterrupt as e:
    print("用户停止: ", e)
except BaseException as e:
    print(f"异常: {e}")
finally:
    # 停止 sensor
    if isinstance(sensor, Sensor):
        sensor.stop()
    # 销毁显示
    Display.deinit()
    os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
    time.sleep_ms(100)
    # 释放媒体缓冲区
    MediaManager.deinit()

  ```

## 2.简单图像处理
+ 这里使用简单的程序来体现
+ 处理使用的模板
+ ```python
    import time
    import os
    import sys

    from media.sensor import *
    from media.display import *
    from media.media import *

    sensor = None

    try:
        print("camera_test")
        sensor = Sensor()
        sensor.reset()
        sensor.set_framesize(Sensor.FHD)
        sensor.set_pixformat(Sensor.YUV420SP)
        bind_info = sensor.bind_info()
        Display.bind_layer(**bind_info, layer=Display.LAYER_VIDEO1)
        sensor.set_framesize(width=640, height=480, chn=CAM_CHN_ID_2)
        sensor.set_pixformat(Sensor.RGB565, chn=CAM_CHN_ID_2)
        Display.init(Display.LT9611, to_ide=True, osd_num=2)
        MediaManager.init()
        sensor.run()

        while True:
            os.exitpoint()
            img = sensor.snapshot(chn=CAM_CHN_ID_2)
            Display.show_image(img, x=1920 - 640, layer=Display.LAYER_OSD1)

    except KeyboardInterrupt as e:
        print("用户停止: ", e)
    except BaseException as e:
        print(f"异常: {e}")
    finally:
        if isinstance(sensor, Sensor):
            sensor.stop()
        Display.deinit()
        os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
        time.sleep_ms(100)
        MediaManager.deinit()
    ```
+ 代码1，使用大津法实现二值化，使用腐蚀膨胀处理图像
  ```python
  import time
    import os
    import sys

    from media.sensor import *
    from media.display import *
    from media.media import *

    sensor = None

    try:
    print("camera_test")
    sensor = Sensor()
    sensor.reset()
    sensor.set_framesize(Sensor.FHD)
    sensor.set_pixformat(Sensor.YUV420SP)
    bind_info = sensor.bind_info()
    Display.bind_layer(**bind_info, layer=Display.LAYER_VIDEO1)
    sensor.set_framesize(width=640, height=480, chn=CAM_CHN_ID_2)
    sensor.set_pixformat(Sensor.RGB565, chn=CAM_CHN_ID_2)
    Display.init(Display.LT9611, to_ide=True, osd_num=2)
    MediaManager.init()
    sensor.run()

    while True:
        os.exitpoint()
        img = sensor.snapshot(chn=CAM_CHN_ID_2)
        img.to_grayscale()
        img.binary([(0,(img.get_histogram().get_threshold().value()))])
        img.erode(2)
        Display.show_image(img, x=1920 - 640, layer=Display.LAYER_OSD1)

    except KeyboardInterrupt as e:
    print("用户停止: ", e)
    except BaseException as e:
    print(f"异常: {e}")
    finally:
    if isinstance(sensor, Sensor):
        sensor.stop()
    Display.deinit()
    os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
    time.sleep_ms(100)
    MediaManager.deinit()

    + 代码2，寻找色块
    ```python
    # Find Blobs Example
    #
    # This example shows off how to find blobs in the image.
    import time, os, gc, sys

    from media.sensor import *
    from media.display import *
    from media.media import *

    DETECT_WIDTH = ALIGN_UP(320, 16)
    DETECT_HEIGHT = 240

    sensor = None

    def camera_init():
        global sensor

        # construct a Sensor object with default configure
        sensor = Sensor(width=DETECT_WIDTH,height=DETECT_HEIGHT)
        # sensor reset
        sensor.reset()
        # set hmirror
        # sensor.set_hmirror(False)
        # sensor vflip
        # sensor.set_vflip(False)

        # set chn0 output size
        sensor.set_framesize(width=DETECT_WIDTH,height=DETECT_HEIGHT)
        # set chn0 output format
        sensor.set_pixformat(Sensor.RGB565)
        # use IDE as display output
        Display.init(Display.VIRT, width= DETECT_WIDTH, height = DETECT_HEIGHT,fps=100,to_ide = True)
        # init media manager
        MediaManager.init()
        # sensor start run
        sensor.run()

    def camera_deinit():
        global sensor
        # sensor stop run
        sensor.stop()
        # deinit display
        Display.deinit()
        # sleep
        os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
        time.sleep_ms(100)
        # release media buffer
        MediaManager.deinit()

    def capture_picture():

        fps = time.clock()
        while True:
            fps.tick()
            try:
                os.exitpoint()
                global sensor
                img = sensor.snapshot()

                # select color
                thresholds = [[0, 80, 40, 80, 10, 80]]      # red
                # thresholds = [[0, 80, -120, -10, 0, 30]]    # green
                # thresholds = [[0, 80, 30, 100, -120, -60]]  # blue
                # find all blobs，and draw rectangles
                blobs=img.find_blobs(thresholds ,pixels_threshold= 500)
                for blob in blobs:
                    img.draw_rectangle(blob[0], blob[1], blob[2], blob[3], color = (255, 255, 0))

                # draw result to screen
                Display.show_image(img)
                img = None

                gc.collect()
                print(fps.fps())
            except KeyboardInterrupt as e:
                print("user stop: ", e)
                break
            except BaseException as e:
                print(f"Exception {e}")
                break

    def main():
        os.exitpoint(os.EXITPOINT_ENABLE)
        camera_is_init = False
        try:
            print("camera init")
            camera_init()
            camera_is_init = True
            print("camera capture")
            capture_picture()
        except Exception as e:
            print(f"Exception {e}")
        finally:
            if camera_is_init:
                print("camera deinit")
                camera_deinit()

    if __name__ == "__main__":
        main()
