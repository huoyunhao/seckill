# 项目简介
本项目是使用SSM框架开发的高并发限时秒杀web应用。
### 项目功能介绍：
- 商品秒杀开启前，用户能看到商品秒杀倒计时，但不能进行秒杀。
 - 商品秒杀开启时，可以进行秒杀但不能进行重复秒杀。
 - 商品秒杀结束后，显示商品秒杀已结束，阻止用户进行秒杀。
# 技术
- 前端技术 ：Bootstrap + jQuery
- 后端技术 ：Spring + SpringMVC + MyBatis
- 依赖管理：Maven
- 版本控制：Git
- 数据库： MySQL + Redis(jedis操作redis)
- 服务器： Tomcat
- 日志工具：logback
- 数据库连接池：cp30
- 序列化：jackson，protostuff
# 特性
- 简易限时秒杀 Web 应用。秒杀开启前显示商品秒杀倒计时；开启时允许用户执行秒杀，但不允许用户重复秒杀同一件商品；结束后显示商品秒杀已结束，并关闭秒杀URL
- 使用 SSM 框架开发后端业务，采用 RESTful 风格接口设计，并使用 JSON 交互数据
- 使用 MD5 混淆秒杀链接，从而防止用户破解秒杀接口提前秒杀
- 使用 Redis 缓存热点数据，减少对数据库的访问，提高页面响应时间
- 使用存储过程执行秒杀操作，减少数据库行级锁时间，提高SQL语句执行速度
# 开发工具
IntelliJ IDEA + MySQL + Git + Chrome+Redis
# 使用说明
- git clone xx或者Download Zip
- 打开IDEA --> File --> New --> Open
- 项目导入后，打开 Project Settings -->Project 设置 Project SDK (本项目JDK版本需在1.8以上)
- 打开File --> Settings --> Build,Execution,Deployment -->Maven 配置maven相关信息
- 在 sql 包下，执行 seckill.sql 与 execute_seckill.sql，建立数据库，然后找到 jdbc.properties 文件修改username and password
### 项目启动过程：
- 启动 MySQL，启动 Redis
- 为项目添加 tomacat 服务器，部署项目并运行
- 打开浏览器进入 http://localhost:8080/seckill/list
# 项目优点
- CDN部署，使用户在不直接访问后台服务器的情况下对静态资源的获取
- 秒杀接口隐藏，防止用户利用脚本恶意刷单
- 使用Redis缓存秒杀商品信息，减低Mysql服务器的压力
- 减少行级锁持有的时间，把事务中减库存的操作放在后面
- 使用存储过程，把简单逻辑在Mysql端执行，屏蔽网络延迟和GC影响
# 后端接口 
## 1     显示秒杀商品信息列表
接口： /seckill/list
备注：从数据库查询到秒杀列表返回给前端
##  2   显示秒杀商品详情
接口：/seckill/{seckillId}/detail     
参数：传入秒杀商品的ID
备注：如果没有秒杀商品则跳转到/seckill/list接口，否则返回秒杀商品详情。
## 3   暴露秒杀端口
接口：/seckill /{seckillId}/exposer
参数：传入秒杀商品的ID
技术点： 
![获取秒杀接口流程](https://upload-images.jianshu.io/upload_images/4157022-131b4c343acdf85a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4   获取系统时间
备注：为了前端进行判断商品是否已经开始秒杀，以服务器时间为准。
## 5   执行秒杀过程
接口： /{seckillId}/{md5}/execution
参数：秒杀商品ID,md5 从cookies中获得用户的手机号作为秒杀用户ID
技术点：
![执行秒杀](https://upload-images.jianshu.io/upload_images/4157022-9e48cf2e1964b863.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
存储过程和数据库sql：
```
-- 数据库初始化脚本

-- 创建数据库
CREATE DATABASE seckill;
-- 使用数据库
use seckill;
CREATE TABLE seckill(
  `seckill_id` BIGINT NOT NUll AUTO_INCREMENT COMMENT '商品库存ID',
  `name` VARCHAR(120) NOT NULL COMMENT '商品名称',
  `number` int NOT NULL COMMENT '库存数量',
  `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `start_time` TIMESTAMP  NOT NULL COMMENT '秒杀开始时间',
  `end_time`   TIMESTAMP   NOT NULL COMMENT '秒杀结束时间',
  PRIMARY KEY (seckill_id),
  key idx_start_time(start_time),
  key idx_end_time(end_time),
  key idx_create_time(create_time)
)ENGINE=INNODB AUTO_INCREMENT=1000 DEFAULT CHARSET=utf8 COMMENT='秒杀库存表';

-- 初始化数据
INSERT into seckill(name,number,start_time,end_time)
VALUES
  ('8000元秒杀iphonex',100,'2019-03-15 00:00:00','2019-03-17 00:00:00'),
  ('3500元秒杀ipad',200,'2019-03-15 00:00:00','2019-03-18 00:00:00'),
  ('18000元秒杀mac book pro',300,'2019-03-28 00:00:00','2019-03-29 00:00:00'),
  ('15000元秒杀iMac',400,'2019-03-15 00:00:00','2019-03-29 00:00:00');

-- 秒杀成功明细表
-- 用户登录认证相关信息(简化为手机号)
CREATE TABLE success_killed(
  `seckill_id` BIGINT NOT NULL COMMENT '秒杀商品ID',
  `user_phone` BIGINT NOT NULL COMMENT '用户手机号',
  `state` TINYINT NOT NULL DEFAULT -1 COMMENT '状态标识:-1:无效 0:成功 1:已付款 2:已发货',
  `create_time` TIMESTAMP NOT NULL COMMENT '创建时间',
  PRIMARY KEY(seckill_id,user_phone),/*联合主键*/
  KEY idx_create_time(create_time)
)ENGINE=INNODB DEFAULT CHARSET=utf8 COMMENT='秒杀成功明细表';

  -- SHOW CREATE TABLE seckill;#显示表的创建信息


--连接数据库控制台
mysql -uroot -p

--为什么手写DDL
--记录每次上线的DDL修改
--上线V1.1
ALTER TABLE seckill
DROP INDEX idx_create_time,
ADD index idx_c_s(start_time,create_time);

--上线V1.2
--DDL
``` 
```
-- 秒杀执行存储过程
DELIMITER $$ -- onsole ; 转换为
$$
-- 定义存储过程
-- 参数：in 输入参数; out 输出参数
-- row_count():返回上一条修改类型sql(delete,insert,upodate)的影响行数
-- row_count: 0:未修改数据; >0:表示修改的行数; <0:sql错误/未执行修改sql
CREATE PROCEDURE `seckill`.`execute_seckill`
  (IN v_seckill_id bigint, IN v_phone BIGINT,
   IN v_kill_time  TIMESTAMP, OUT r_result INT)
  BEGIN
    DECLARE insert_count INT DEFAULT 0;
    START TRANSACTION;
    INSERT ignore INTO success_killed (seckill_id, user_phone, create_time)
    VALUES (v_seckill_id, v_phone, v_kill_time);
    SELECT ROW_COUNT() INTO insert_count;
    IF (insert_count = 0)
    THEN
      ROLLBACK;
      SET r_result = -1;
    ELSEIF (insert_count < 0)
      THEN
        ROLLBACK;
        SET r_result = -2;
    ELSE
      UPDATE seckill
      SET number = number - 1
      WHERE seckill_id = v_seckill_id
        AND end_time > v_kill_time
        AND start_time < v_kill_time
        AND number > 0;
      SELECT ROW_COUNT() INTO insert_count;
      IF (insert_count = 0)
      THEN
        ROLLBACK;
        SET r_result = 0;
      ELSEIF (insert_count < 0)
        THEN
          ROLLBACK;
          SET r_result = -2;
      ELSE
        COMMIT;
        SET r_result = 1;
      END IF;
    END IF;
  END;
$$
-- 代表存储过程定义结束

DELIMITER ;

SET @r_result = -3;
-- 执行存储过程
call execute_seckill(1001, 13631231234, now(), @r_result);
-- 获取结果
SELECT @r_result;

-- 存储过程
-- 1.存储过程优化：事务行级锁持有的时间
-- 2.不要过度依赖存储过程
-- 3.简单的逻辑可以应用存储过程
-- 4.QPS:一个秒杀单6000/qps
```
**先插入订单，再更新库存，作为一次事务提交。成功则提交，失败则回滚。**
在success_killed表中，用户手机号和seckill_id作为联合主键，防止一个用户抢购多个相同商品，属于数据库约束。
# 前端逻辑
-  1，当用户点击秒杀商品详情时访问  **显示秒杀商品详情** 接口获取商品详情,从cookies获取用户手机号，没有则让用户填写，访问**获取系统时间**接口，获取系统时间，进行时间判断，
---|1.1如果秒杀未开始，则显示倒计时，
  ------|1.1.1倒计时结束回调**获取秒杀地址接口**
  ---------|1.1.1.2 接口未开启 显示倒计时(浏览器时间偏差)
  ---------|1.1.1.2 接口已经开启，显示秒杀按钮，用户点击执行秒杀。获取结果并显示。
---|1.2如果秒杀结束，则显示秒杀结束。
---|1.3如果正在进行，则**获取秒杀地址接口**，如果接口已经开启，显示秒杀按钮，用户点击执行秒杀。获取结果并显示。
![秒杀前端逻辑](https://upload-images.jianshu.io/upload_images/4157022-8e2580b8e245ab36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 项目截图
![秒杀列表](https://upload-images.jianshu.io/upload_images/4157022-c0cf905e0dbf2102.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![秒杀商品](https://upload-images.jianshu.io/upload_images/4157022-5cf85ac65da6cb40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![执行秒杀](https://upload-images.jianshu.io/upload_images/4157022-310308ab2ffc56fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#  Github&&Blog地址
[Github](https://github.com/ayhyh/seckill.git)
[Blog]([https://www.jianshu.com/p/1d52ee68cc4d](https://www.jianshu.com/p/1d52ee68cc4d))







 
