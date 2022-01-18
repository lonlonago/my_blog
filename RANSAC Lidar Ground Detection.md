RANSAC Lidar Ground Detection (1)

Using the random sampling consistency method for ground detection, the main idea is to continuously randomly select three points to construct a plane equation. If the plane contains enough points, then it is considered to be the ground;



The code is as follows. The happy thing is that after optimization, the python code actually runs faster than the c++ part of the code. It only takes 20ms to find the plane, and the matrix operation is really awesome.


````python

import numpy as np
import time
import random
import math

def my_ransac_v3(data,
              distance_threshold=0.3,
              P=0.99,
              sample_size=3,
              max_iterations=10000,
              ):
    """

    :param data:  n*3
    :param sample_size:
    :param P :
    :param distance_threshold:
    :param max_iterations:
    :return:
    """
    # np.random.seed(12345)
    random.seed(12345)
    max_point_num = -999
    i = 0
    K = 10
    L_data = len(data)
    R_L = range(L_data)


    while i < K:
        # 随机选3个点  np.random.choice 很耗费时间，改为random模块
        # s3 = np.random.choice(L_data, sample_size, replace=False)
        s3 = random.sample(R_L, sample_size)

        if abs(data[s3[0],1] - data[s3[1],1]) < 3:
            continue



        # 计算平面方程系数
        coeffs = estimate_plane(data[s3,:], normalize=False)
        if coeffs is None:
            continue
        # 法向量的模, 如果系数标准化了就不需要除以法向量的模了
        r = np.sqrt(coeffs[0]**2 + coeffs[1]**2 + coeffs[2]**2 )
        # 计算每个点和平面的距离，根据阈值得到距离平面较近的点数量
        d = np.divide(np.abs(np.matmul(coeffs[:3], data.T) + coeffs[3]) , r)
        # d = abs(np.matmul(coeffs[:3], data.T) + coeffs[3]) / r
        d_filt = np.array(d < distance_threshold)

        near_point_num = np.sum(d_filt,axis=0)


        if near_point_num > max_point_num:
            max_point_num = near_point_num
            # TODO 这是是否还有一个重新拟合模型的步骤，
            #  有的话怎么用多个点求一个平面
            best_model = coeffs
            best_filt = d_filt

            # 最大迭代次数K，是P的函数, 因为迭代次数足够多，就一定能找到最佳的模型，
            # P是模型提供理想需要结果的概率，也可以理解为模型的点都是内点的概率？
            # P = 0.99
            # 当前模型，所有点中随机抽取一个点，它是内点的概率
            w = near_point_num / L_data
            # np.power(w, 3)是随机抽取三个点，都是内点的概率
            # 1-np.power(w, 3)，就是三个点至少有一个是外点的概率，
            # 也就是得到一个坏模型的概率;
            # 1-P 代表模型永远不会选出一个3个点都是内点的集合的概率，
            # 也就是至少有一个外点；
            # K 就是得到的需要尝试多少次才会得到当前模型的理论次数
            # TODO 完善对该理论最大论迭代次数的理解
            wn = np.power(w, 3)
            p_no_outliers = 1.0 - wn
            # sd_w = np.sqrt(p_no_outliers) / wn
            K = (np.log(1-P) / np.log(p_no_outliers)) #+ sd_w
        # print('# K:', i, K, near_point_num)

        i += 1

        if i > max_iterations:
            print(' RANSAC reached the maximum number of trials.')
            break

    print('took iterations:', i+1, 'best model:', best_model,
          'explains:', max_point_num)
    return np.argwhere(best_filt).flatten(), best_model



def estimate_plane(xyz, normalize=True):
    """
    已知三个点，根据法向量求平面方程；
    返回平面方程的一般形式的四个系数
    :param xyz:  3*3 array
    x1 y1 z1
    x2 y2 z2
    x3 y3 z3
    :return: a b c d

      model_coefficients.resize (4);
      model_coefficients[0] = p1p0[1] * p2p0[2] - p1p0[2] * p2p0[1];
      model_coefficients[1] = p1p0[2] * p2p0[0] - p1p0[0] * p2p0[2];
      model_coefficients[2] = p1p0[0] * p2p0[1] - p1p0[1] * p2p0[0];
      model_coefficients[3] = 0;
      // Normalize
      model_coefficients.normalize ();
      // ... + d = 0
      model_coefficients[3] = -1 * (model_coefficients.template head<4>().dot (p0.matrix ()));

    """
    vector1 = xyz[1,:] - xyz[0,:]
    vector2 = xyz[2,:] - xyz[0,:]

    # 共线性检查 ; 0过滤
    if not np.all(vector1):
        # print('will divide by zero..', vector1)
        return None
    dy1dy2 = vector2 / vector1
    # 2向量如果是一条直线，那么必然它的xyz都是同一个比例关系
    if  not ((dy1dy2[0] != dy1dy2[1])  or  (dy1dy2[2] != dy1dy2[1])):
        return None


    a = (vector1[1]*vector2[2]) - (vector1[2]*vector2[1])
    b = (vector1[2]*vector2[0]) - (vector1[0]*vector2[2])
    c = (vector1[0]*vector2[1]) - (vector1[1]*vector2[0])
    # normalize
    if normalize:
        # r = np.sqrt(a ** 2 + b ** 2 + c ** 2)
        r = math.sqrt(a ** 2 + b ** 2 + c ** 2)
        a = a / r
        b = b / r
        c = c / r
    d = -(a*xyz[0,0] + b*xyz[0,1] + c*xyz[0,2])
    # return a,b,c,d
    return np.array([a,b,c,d])

````


The core of the optimization speed has two points. One is that the distance from the point to the plane is replaced by the matrix operation instead of the loop calculation; the other is that the random module is used for random sampling instead of np.random, and the random module of numpy does not know why it is so slow;



There are two disadvantages of this method to find the plane. One is that the wall is regarded as the ground, and the other is that there is nothing you can do about the slope. I tried the kitti data, and there will be 1 in about 100. In the first case, there will be 10 or so. the second case;



questions to contract me : lonlonago@foxmail.com
