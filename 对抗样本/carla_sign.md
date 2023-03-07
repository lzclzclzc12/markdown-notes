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

- 加入了贴图后：sign3.obj ， new_faces6.txt ， 

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

- 

