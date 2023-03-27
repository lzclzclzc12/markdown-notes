# 普通牌子的设置

- sign6.obj

- masks2

- train_label_new2

- train4

- new_faces3.txt

- ```
  cam_trans[0][2] -= 1
  scale = .52
  ```

# 悬挂牌子的设置

- masks3

- sign12.obj

- new_facas4.txt

- **加入了贴图后**：sign3.obj ， new_faces6.txt ， 

- ```
  专项（左边）：cam_x = 6  cam_y = 6  cam_z = -1.5  yaw = -90  pitch = 0
  cam_trans[0][1] += 3
  cam_trans[0][2] -= 1
  scale = .6
  右边：cam_trans[0][1] -= 3
  ```

- ```
  普通：
  scale = .34
  cam_trans[0][0] += 3.
  ```

- ```
  测试：
  第一组：
  cam_y_list = np.arange(-14.0, -9.0, 0.1)
  transform = carla.Transform(carla.Location(x=65.58855469, y = 147.30387695, z = 3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=-90, roll=0))
  cam_x = cam_y_list[i]
  cam_y = -6
  cam_z = -1.5
  yaw = 0
  
  第二组：
  cam_y_list = np.arange(-9.0, -4.0, 0.1)
  transform = carla.Transform(carla.Location(x = 92.98157227 , y = 41.35225586 , z = 3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0.0, roll=0.000000))
  cam_x = 6
  cam_y = cam_y_list[i]
  cam_z = -1.5
  yaw = 90
  
  第三组：
  cam_y_list = np.arange(-9.0, -4.0, 0.1)
  transform = carla.Transform(carla.Location(x = 92.98157227 , y = 85.8359668 , z = 3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0, roll=0))
  cam_x = 6
  cam_y = cam_y_list[i]
  cam_z = -1.5
  yaw = 90
  
  第四组：
  cam_y_list = np.arange(-12.0, -7.0, 0.1)
  transform = carla.Transform(carla.Location(x = -23.0154541 , y = 34.4849292 , z = 3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=-90, roll=0))
  cam_x = cam_y_list[i]
  cam_y = -6
  cam_z = -1.5
  yaw = 0
  
  
  ```

  ```
  新一轮测试：
  第一组（大约在11.5米处开始攻击）：
  cam_y_list = np.arange(-15.0, -7.5, 0.1)
  transform = carla.Transform(carla.Location(x = 92.98157227 , y = 41.35225586 , z = 3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0.0, roll=0.000000))
  cam_x = 6
  cam_y = cam_y_list[i]
  cam_z = -1.5
  yaw = 90
  
  第二组（大约在15.5米处开始攻击）：
  cam_y_list = np.arange(-18.0, -11, 0.1)
  transform = carla.Transform(carla.Location(x=92.98157227, y=41.35225586, z=3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0.0, roll=0.000000))
  cam_x = 9
  cam_y = cam_y_list[i]
  cam_z = -1.5
  yaw = 90
  
  第三组（大约在17米处开始攻击）：
  cam_y_list = np.arange(-20.0, -12.5, 0.1)
  transform = carla.Transform(carla.Location(x=50.83341797, y=-49.30177734, z=2.59450348),
                              carla.Rotation(pitch=0.000000, yaw=-90, roll=0.000000))
  cam_x = cam_y_list[i]
  cam_y = -8.5
  cam_z = -1.0
  yaw = 0
  pitch = 0
  
  
  ```

  ```
  重新训练之后的一轮：
  第一组（高处）：
  z = 6.11036499
  transform = carla.Transform(carla.Location(x=53.11345703, y=-49.04263184, z = z),
                              carla.Rotation(pitch=0.000000, yaw=-90, roll=0.000000))
  cam_y_list = np.arange(-20.0, -12, 0.1)
  cam_x = cam_y_list[i]
  cam_y = -8.5
  cam_z = 1.6 - sampled_batch['veh_trans'][0][2]
  yaw = 0
  pitch = 0
  
  
  ```

- ```
  数据集
  camera_bp.set_attribute('image_size_x', '1600')
  camera_bp.set_attribute('image_size_y', '1600')
  camera_bp.set_attribute('fov', '90')
  
  第一组（右上方）：
  transform = carla.Transform(carla.Location(x=92.98157227, y=41.35225586, z=3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0.0, roll=0.000000))
  cam_x = random.uniform(6 , 10)
  cam_y = random.uniform(-18 , -11)
  cam_z = random.uniform(-1.5 , -2)
  yaw = 90
  pitch = random.uniform(0, 20)
  
  第二组（正面）：
  transform = carla.Transform(carla.Location(x=92.98157227, y=41.35225586, z=3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0.0, roll=0.000000))
  cam_x = random.uniform(6 , 15)
  cam_y = random.uniform(-10 , 10)
  cam_z = random.uniform(-2 , -1)
  yaw = 180 + math.degrees(math.atan(cam_y / cam_x))
  pitch = random.uniform(0, 20)
  
  第三组（左上）：
  transform = carla.Transform(carla.Location(x=92.98157227, y=52.32907227, z=3.13194611),
                              carla.Rotation(pitch=0.000000, yaw=0.0, roll=0.000000))
  cam_x = random.uniform(6 , 10)
  cam_y = random.uniform(11 , 18)
  cam_z = random.uniform(-1.5 , -2)
  yaw = -90
  pitch = random.uniform(0, 20)
  
  第四组（高一点）：
  z = random.uniform(6 , 8)
  transform = carla.Transform(carla.Location(x=92.98157227, y=41.35225586, z = z),
                              carla.Rotation(pitch=0.000000, yaw=0, roll=0.000000))
  cam_x = random.uniform(6 , 10)
  cam_y = random.uniform(-18 , -11)
  cam_z = random.uniform(1.1 - sampled_batch['veh_trans'][0][2] , 1.6 - sampled_batch['veh_trans'][0][2])
  yaw = 90
  pitch = random.uniform(0, 20)
  
  第五组（近一点）：
  z = 3.13194611
  transform = carla.Transform(carla.Location(x=92.98157227, y=41.35225586, z = z),
                              carla.Rotation(pitch=0.000000, yaw=0, roll=0.000000))
  cam_x = 6
  cam_y = random.uniform(-11 , -7)
  cam_z = random.uniform(1.1 - sampled_batch['veh_trans'][0][2] , 1.6 - sampled_batch['veh_trans'][0][2])
  yaw = 90
  pitch = random.uniform(0, 20)
  ```


# 可转移性测试

- faster rcnn: 当测试集中包括**右上角常规牌子**（最常见的攻击位置，牌子在摄像头右上角，高度为3-4米）、**正面牌子**（牌子在摄像头正面）、**右上角高处牌子**（牌子在摄像头右上角，高度为6-8米）时，正确率为50.8%；当测试集中只包括**右上角常规牌子**时，正确率为86.6%
- ssd：当测试集中包括**右上角常规牌子**（牌子在摄像头右上角，高度为3-4米）、**正面牌子**（牌子在摄像头正面）、**右上角高处牌子**（牌子在摄像头右上角，高度为6-8米）时，正确率为42.5%；当测试集中只包括**右上角常规牌子**时，正确率为36.6%；可能与ssd本身的性能差有关系
- yolov5x: 当测试集中包括**右上角常规牌子**（牌子在摄像头右上角，高度为3-4米）、**正面牌子**（牌子在摄像头正面）、**右上角高处牌子**（牌子在摄像头右上角，高度为6-8米）时，正确率为83.4%；当测试集中只包括**右上角常规牌子**时，正确率为98%；

- sparse rcnn: 当测试集中包括**右上角常规牌子**（最常见的攻击位置，牌子在摄像头右上角，高度为3-4米）、**正面牌子**（牌子在摄像头正面）、**右上角高处牌子**（牌子在摄像头右上角，高度为6-8米）时，正确率为42.7%；当测试集中只包括**右上角常规牌子**时，正确率为84%
- yolov8x: 当测试集中包括**右上角常规牌子**（最常见的攻击位置，牌子在摄像头右上角，高度为3-4米）、**正面牌子**（牌子在摄像头正面）、**右上角高处牌子**（牌子在摄像头右上角，高度为6-8米）时，正确率为86.8%；当测试集中只包括**右上角常规牌子**时，正确率为97%

# 纸袋广告

## 数据集

- ```
  45 45
  第一组（正面）300张
  z = random.uniform(0.16 , 0.8)
  transform = carla.Transform(carla.Location(x=114.01196289, y=51.25675293, z=z),
                              carla.Rotation(pitch=0.000000, yaw=90, roll=0.000000))
                              
  transform = carla.Transform(carla.Location(x=94.54625, y=41.60850586, z=z),
                                          carla.Rotation(pitch=0.000000, yaw=-90, roll=0.000000))
  cam_x = -4
  cam_x = 4
  cam_y = random.uniform(4.5 , 10)
  cam_y = random.uniform(-10 , -4.5)
  
  cam_x = -6
  cam_y = random.uniform(6.5 , 10)
  
  cam_x = -7
  cam_y = random.uniform(7.5 , 10)
  
  cam_z =  random.uniform(1.5 , 2.0) - z
  yaw = -90
  yaw = 90
  pitch = 0
  
  第二组（侧面）
  
  ```

- ```
  测试集：
  第一组：
  cam_y_list = np.arange(-10, -4.5, 0.1)
  z = random.uniform(0.16 , 0.30)
  transform = carla.Transform(carla.Location(x=94.47762695, y=42.58241211, z=z),
                              carla.Rotation(pitch=0.000000, yaw=-90, roll=0.000000))
                              
  cam_x = 4
  cam_y = cam_y_list[i]
  cam_z =  random.uniform(1.5 , 2.0) - z
  yaw = 90
  
  transform = carla.Transform(carla.Location(x=94.47762695, y=42.58241211, z=z),
                              carla.Rotation(pitch=0.000000, yaw=-90, roll=0.000000))
  
  
  第二组：
  z = random.uniform(0.16 , 0.35)
  transform = carla.Transform(carla.Location(x=94.08273438, y=86.57958984, z=z),
                              carla.Rotation(pitch=0.000000, yaw=yaw1, roll=0.000000))
  cam_x = 4
  cam_y = cam_y_list[i]
  cam_z =  random.uniform(1.5 , 2.0) - z
  yaw = 90
  
  
  第三组：
  z = random.uniform(0.16, 0.30)
  yaw1 = 180
  transform = carla.Transform(carla.Location(x=52.38172852, y=145.96547852, z=z),
                              carla.Rotation(pitch=0.000000, yaw=yaw1, roll=0.000000))
  cam_x = -random.uniform(4.5 , 5)
  cam_y = -4
  yaw = 0
  ```

## 测试

- ```
  mm_3  D:\PyCharmProject\DualAttentionAttack-main\src\contents1\zhidai\7.jpg
  test1: 横向4.5-5m，纵向4m，高度0.16-0.3,78.5%
  test2: 横向4.5-5m，纵向4m，高度0.3-0.8,65%
  test3: 横向4.5-5m，纵向5m，高度0.3-0.8,52.5%
  test4: 横向3-4m，纵向4m，高度0.16-0.3, 87%
  
  ```

  