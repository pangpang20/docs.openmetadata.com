# 在华为云部署项目

## Demo介绍

开发任务中对Demo的要求如下：

- Openmetadata 安装成功后使用GaussDB 存储其采集到的元数据信息。
- 可以通过 openmetadata WebUI 管理存储在GaussDB内的元数据信息。

经过分析，采用在GaussDB中创建样例数据库，并添加一些表，视图，存储过程等对象。添加测试数据，然后通过openmetadata采集元数据信息。

### Demo部署架构
下图是Demo的部署架构，基于CCE容器化部署：

![部署架构](../assets/3-1.png)

1.**终端层**

支持不同的类型终端访问Web UI，例如浏览器、手机等

2.**网关层**

通过ELB进行负载均衡代理

3.**中间层**

在CCE中部署3个无状态服务：

- openmetadata-server: 负责提供openmetadata的Web UI
- openmetadata-ingestion: 负责采集元数据信息
- execute-migrate-all: 负责执行数据库迁移脚本

在CCE中部署2个有状态服务：

- openmetadata-mysql: 负责存储openmetadata的元数据信息
- openmetadata-elasticsearch: 负责存储openmetadata的元数据信息索引

4.**数据库层**

在GaussDB中创建样例数据库，并添加一些表、视图、存储过程等对象。

## 步骤一：云资源购买与配置

Demo部署主要依赖的云资源如下：

- VPC: 虚拟私有云，实现隔离
- ECS：制作镜像
- ELB：负载均衡和代理
- GaussDB：数据库加工原始数据
- CCE：容器化部署
- SWR: 存放容器镜像
- NAT网关: 容器中访问公网


### 购买ECS并克隆项目

ECS配置如下：

- 购买数量：1台
- 规格：鲲鹏通用计算增强型 |  kc1.2xlarge.4 | 8vCPUs | 32GiB
- 镜像: Huawei Cloud EulerOS 2.0 标准版 64位 ARM版
- 登录凭证：密码
- 系统盘: 通用型SSD, 40GiB

具体购买操作可参考 [快速购买和使用Linux ECS](https://support.huaweicloud.com/qs-ecs/ecs_01_0103.html)

购买后，待服务器启动，登录服务器按下面的操作继续。

#### 安装Docker

参考 [基于EulerOS部署Docker](https://bbs.huaweicloud.com/blogs/441849)

#### 安装Git

参考 [基于EulerOS配置GitHub](https://bbs.huaweicloud.com/blogs/441850)

#### 克隆项目并且制作镜像


```bash
# 克隆项目
cd /opt/cyl
git clone git@github.com:pangpang20/OpenMetadata.git
cd OpenMetadata


# 创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate

# 环境要求：
Docker 20 or higher
Java JDK 17
Antlr 4.9.2 - sudo make install_antlr_cli
JQ - brew install jq (osx) apt-get install jq (Ubuntu)
Maven 3.5.x or higher - (with Java JDK 11)
Python 3.8 or 3.9
Node 18.x
Yarn ^1.22.0
Rpm (Optional, only to run RPM profile with maven)

# 编译镜像
sh docker/run_local_docker.sh

```


#### 上传镜像到SWR

进入`容器镜像服务 SWR`，点击 `组织管理` ,点击 `创建组织`

点击`总览`，点击右上角 `登录指令`，复制到ECS中执行

上传镜像


```bash
sudo docker tag {镜像名称}:{版本名称} swr.cn-south-1.myhuaweicloud.com/{组织名称}/{镜像名称}:{版本名称}

sudo docker push swr.cn-south-1.myhuaweicloud.com/{组织名称}/{镜像名称}:{版本名称}

```

上传后在`我的镜像`中可以查看


### 购买GaussDB并创建用户和数据库

GaussDB配置如下：

- 数据库引擎: GaussDB
- 数据库引擎版本:V2.0-8.201
- 内核引擎版本:505.2.0
- 实例类型: 集中式版
- 部署形态: 1主2备
- 性能规格: 通用型（1:4）| 4 vCPUs | 16 GB
- 存储空间: 40 GB
- 数据库端口: 默认端口8000

具体购买操作可参考 [购买并通过界面化工具DAS连接GaussDB实例（推荐）](https://support.huaweicloud.com/qs-gaussdb/gaussdb_01_622.html)

购买后，待数据库实例启动，登录DAS创建数据库和创建用户。

> **🔔 注意：**
> 这里需要记录EIP或内网IP，数据库名，用户名，密码，后面需要。

#### 样例sql

在DAS中执行以下SQL，创建数据库和表等对象。

```sql
创建数据库：metadb
--1.创建表
DROP TABLE IF EXISTS  products;
DROP TABLE IF EXISTS  categories;
DROP TABLE IF EXISTS  sales;
DROP TABLE IF EXISTS  customers;
DROP TABLE IF EXISTS  inventory;



-- 商品类别表
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,  -- Category ID
    name VARCHAR(255) NOT NULL        -- Category Name
);

-- 添加描述
COMMENT ON TABLE categories IS 'Table storing product categories';
COMMENT ON COLUMN categories.category_id IS 'The unique identifier for each category';
COMMENT ON COLUMN categories.name IS 'The name of the product category';

-- 商品表
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,  -- Product ID
    name VARCHAR(255) NOT NULL,     -- Product Name
    description TEXT,               -- Product Description
    price DECIMAL(10, 2) NOT NULL,  -- Product Price
    category_id INT,                -- Category ID
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP  -- Creation Time
);

-- 添加描述
COMMENT ON TABLE products IS 'Table storing product information';
COMMENT ON COLUMN products.product_id IS 'The unique identifier for each product';
COMMENT ON COLUMN products.name IS 'The name of the product';
COMMENT ON COLUMN products.description IS 'A description of the product';
COMMENT ON COLUMN products.price IS 'The price of the product';
COMMENT ON COLUMN products.category_id IS 'The category ID of the product';
COMMENT ON COLUMN products.created_at IS 'The creation timestamp of the product record';

-- 销售记录表
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,      -- Sale ID
    product_id INT NOT NULL,         -- Product ID
    customer_id INT NOT NULL,        -- Customer ID
    sale_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP, -- Sale Date
    quantity INT NOT NULL,           -- Quantity Sold
    total DECIMAL(10, 2) NOT NULL    -- Total Sale Amount
);

-- 添加描述
COMMENT ON TABLE sales IS 'Table storing sales transactions';
COMMENT ON COLUMN sales.sale_id IS 'The unique identifier for each sale transaction';
COMMENT ON COLUMN sales.product_id IS 'The ID of the product being sold';
COMMENT ON COLUMN sales.customer_id IS 'The ID of the customer making the purchase';
COMMENT ON COLUMN sales.sale_date IS 'The date and time the sale occurred';
COMMENT ON COLUMN sales.quantity IS 'The quantity of the product sold';
COMMENT ON COLUMN sales.total IS 'The total sale amount for the transaction';

-- 客户信息表
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,  -- Customer ID
    name VARCHAR(255) NOT NULL,       -- Customer Name
    email VARCHAR(255) UNIQUE,        -- Customer Email
    phone VARCHAR(15)                -- Customer Phone
);

-- 添加描述
COMMENT ON TABLE customers IS 'Table storing customer information';
COMMENT ON COLUMN customers.customer_id IS 'The unique identifier for each customer';
COMMENT ON COLUMN customers.name IS 'The name of the customer';
COMMENT ON COLUMN customers.email IS 'The email address of the customer';
COMMENT ON COLUMN customers.phone IS 'The phone number of the customer';

-- 库存表
CREATE TABLE inventory (
    product_id INT PRIMARY KEY,       -- Product ID
    stock_quantity INT NOT NULL       -- Stock Quantity
);

-- 添加描述
COMMENT ON TABLE inventory IS 'Table storing inventory data';
COMMENT ON COLUMN inventory.product_id IS 'The ID of the product in the inventory';
COMMENT ON COLUMN inventory.stock_quantity IS 'The available stock quantity of the product';

--2.插入测试数据

-- 插入商品类别数据
INSERT INTO categories (name) VALUES
('Electronics'),
('Clothing'),
('Books'),
('Food');

-- 插入商品数据
INSERT INTO products (name, description, price, category_id) VALUES
('Laptop', 'High performance laptop', 999.99, 1),
('T-shirt', 'Cotton T-shirt', 19.99, 2),
('Novel', 'A gripping mystery novel', 12.99, 3),
('Apple', 'Fresh organic apples', 2.99, 4),
('Headphones', 'Noise-cancelling headphones', 149.99, 1);

-- 插入客户数据
INSERT INTO customers (name, email, phone) VALUES
('Alice', 'alice@example.com', '123-456-7890'),
('Bob', 'bob@example.com', '234-567-8901'),
('Charlie', 'charlie@example.com', '345-678-9012');

-- 插入库存数据
INSERT INTO inventory (product_id, stock_quantity) VALUES
(1, 50),
(2, 200),
(3, 100),
(4, 500),
(5, 30);

-- 插入销售记录数据
INSERT INTO sales (product_id, customer_id, quantity, total) VALUES
(1, 1, 2, 1999.98),
(2, 2, 3, 59.97),
(3, 3, 1, 12.99),
(4, 1, 10, 29.90),
(5, 2, 1, 149.99);

COMMIT;

--3.创建存储过程
-- 1. 创建存储过程：add_sale
CREATE OR REPLACE PROCEDURE add_sale(
    p_product_id INT,
    p_customer_id INT,
    p_quantity INT
)
AS
BEGIN
    -- 计算销售总金额
    DECLARE
        v_total DECIMAL(10, 2);
    BEGIN
        SELECT price * p_quantity INTO v_total
        FROM products
        WHERE product_id = p_product_id;

        -- 插入销售记录
        INSERT INTO sales (product_id, customer_id, quantity, total)
        VALUES (p_product_id, p_customer_id, p_quantity, v_total);

        -- 更新库存
        UPDATE inventory
        SET stock_quantity = stock_quantity - p_quantity
        WHERE product_id = p_product_id;
    END;
END;


-- 2. 创建存储过程：add_product
CREATE OR REPLACE PROCEDURE add_product(
    p_product_name VARCHAR,
    p_price DECIMAL(10, 2),
    p_stock_quantity INT
)
AS
BEGIN
    -- 插入产品记录
    INSERT INTO products (product_name, price, stock_quantity)
    VALUES (p_product_name, p_price, p_stock_quantity);
END;
/

-- 3. 创建存储过程：update_product_price
CREATE OR REPLACE PROCEDURE update_product_price(
    p_product_id INT,
    p_new_price DECIMAL(10, 2)
)
AS
BEGIN
    -- 更新产品价格
    UPDATE products
    SET price = p_new_price
    WHERE product_id = p_product_id;
END;
/

-- 4. 创建存储过程：add_customer
CREATE OR REPLACE PROCEDURE add_customer(
    p_customer_name VARCHAR,
    p_email VARCHAR
)
AS
BEGIN
    -- 插入客户记录
    INSERT INTO customers (customer_name, email)
    VALUES (p_customer_name, p_email);
END;
/

-- 5. 创建存储过程：add_inventory
CREATE OR REPLACE PROCEDURE add_inventory(
    p_product_id INT,
    p_additional_quantity INT
)
AS
BEGIN
    -- 更新库存
    UPDATE inventory
    SET stock_quantity = stock_quantity + p_additional_quantity
    WHERE product_id = p_product_id;
END;
/

CREATE OR REPLACE FUNCTION get_total_sales(p_product_id INT)
RETURNS DECIMAL AS $$
DECLARE
    v_total_sales DECIMAL(10, 2);
BEGIN
    -- 计算商品的总销售额
    SELECT SUM(total) INTO v_total_sales
    FROM sales
    WHERE product_id = p_product_id;

    -- 返回总销售额，如果没有销售记录则返回 0
    RETURN COALESCE(v_total_sales, 0);
END;
$$ LANGUAGE plpgsql;

-- 添加描述
COMMENT ON FUNCTION get_total_sales IS 'Function for calculating the total sales amount for a given product';

CREATE OR REPLACE FUNCTION get_customer_purchase_history(p_customer_id INT)
RETURNS TABLE(product_name VARCHAR, quantity INT, total DECIMAL) AS $$
BEGIN
    -- 返回客户购买的商品信息
    RETURN QUERY
    SELECT p.name, s.quantity, s.total
    FROM sales s
    JOIN products p ON s.product_id = p.product_id
    WHERE s.customer_id = p_customer_id;
END;
$$ LANGUAGE plpgsql;

-- 添加描述
COMMENT ON FUNCTION get_customer_purchase_history IS 'Function for retrieving the purchase history of a customer';


--4.创建视图
-- 商品销售详情
CREATE VIEW product_sales_view AS
SELECT p.name AS product_name,
       c.name AS category_name,
       SUM(s.quantity) AS total_quantity_sold,
       SUM(s.total) AS total_sales
FROM sales s
JOIN products p ON s.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
GROUP BY p.name, c.name;

--客户购买历史
CREATE VIEW customer_purchase_history_view AS
SELECT c.name AS customer_name,
       p.name AS product_name,
       s.quantity,
       s.total,
       s.sale_date
FROM sales s
JOIN products p ON s.product_id = p.product_id
JOIN customers c ON s.customer_id = c.customer_id;


--查询商品销售详情：
SELECT * FROM product_sales_view;


--查询客户购买历史：
SELECT * FROM customer_purchase_history_view WHERE customer_name = 'Alice';

--获取商品的总销售额：
SELECT get_total_sales(1);

--获取客户的购买历史：
SELECT * FROM get_customer_purchase_history(1);
```


### 购买CCE和节点


CCE配置如下：

- 集群类型CCE: Turbo
- 容器网络模型云原生网络: 2.0
- 集群版本: v1.30
- 集群规模: 50 节点
- 集群 master 实例数: 3实例（高可用）

CCE集群创建后，创建节点，节点配置如下：

- 节点类型：弹性云服务器-虚拟机
- 节点规格：鲲鹏通用计算增强型 | kc2.2xlarge.4 | 8 vCPUs | 32 GiB
- 容器引擎：Docker
- 操作系统：Huawei Cloud EulerOS2.0
- 登录方式：密码
- 磁盘：默认
- 节点数量：3

具体购买操作可参考 [在CCE集群中部署NGINX无状态工作负载](https://support.huaweicloud.com/qs-cce/cce_qs_0003.html)


### 购买ELB

ELB配置如下：

- 实例类型：独享型
- 实例规格：弹性规格，应用型+网络型
- 所属VPC：和CCE在同一个VPC
- 弹性公网IP带宽：10 Mbit/s

具体购买操作可参考 [实现单个Web应用的负载均衡](https://support.huaweicloud.com/qs-elb/zh-cn_topic_0052569751.html)

## 步骤二：部署Demo

可以参考 `OpenMetadata/docker/development/docker-compose.yml` 来配置相关参数

### 有状态的工作负载

在CCE中添加2个有状态的工作负载，具体如下：

#### openmetadata-mysql

负责存储openmetadata的元数据信息


#### openmetadata-elasticsearch

负责存储openmetadata的元数据信息索引

部署后如下：

![](../assets/3-2.png)

### 无状态的工作负载

在CCE中添加3个无状态的工作负载，具体如下：

#### openmetadata-server

负责提供openmetadata的Web UI

#### openmetadata-ingestion

负责采集元数据信息

#### execute-migrate-all

负责执行数据库迁移脚本

部署后如下：

![](../assets/3-3.png)


## 步骤三：访问UI

### 访问GaussDB

基于前面的demo SQL，我们在GaussDB中执行SQL语句，查看数据：

![](../assets/4-1.png)

表对象：

![](../assets/4-2.png)

视图对象：

![](../assets/4-3.png)

存储过程对象：

![](../assets/4-4.png)

接下来，我们通过openmetadata采集这些元数据信息。

### 访问airflow

在浏览器访问ELB的公网IP，访问`airflow`地址：

![](../assets/5-1.png)

输入用户名和密码（admin，admin），进入`airflow`界面：

![](../assets/5-2.png)

这时候还没有DAG任务，我们接下来通过在`openmetadata-server`创建DAG任务。


### 访问openmetadata-server

在浏览器访问ELB的公网IP，访问`openmetadata-server`地址：

![](../assets/6-1.png)

输入用户名和密码（admin@open-metadata.org，admin），进入`openmetadata-server`界面：

按下图点击 `Settings`：

![](../assets/6-2.png)

按下图点击 `Services`：

![](../assets/6-3.png)

按下图点击 `Databases`：

![](../assets/6-4.png)

按下图点击 `Add new service`：

![](../assets/6-5.png)

按下图点击 `Gaussdb`, 然后点击`Next`：
![](../assets/6-6.png)

按下图填写相关信息，然后点击`Next`：
![](../assets/6-7.png)

按下图填写相关信息，然后点击`Save`：
![](../assets/6-8.png)

按下图点击 `Add Ingestion`:

![](../assets/6-9.png)

按下图填写相关信息，然后点击`Next`:

![](../assets/6-10.png)

配置手工执行，然后点击`Add & Deploy`：

![](../assets/6-11.png)

在这里再添加两个ingestion:

![](../assets/6-12.png)

一个是采集血缘信息：

![](../assets/6-13.png)

![](../assets/6-14.png)

一个是采集剖析信息：

![](../assets/6-15.png)

![](../assets/6-16.png)

添加完成后，这里可以看到3个ingestion：

- 先点击Type为metadata的ingestion，然后点击`Run`：
- 运行完成后，再点击Type为profiler的ingestion，然后点击`Run`：
- 运行完成后，再点击Type为lineage的ingestion，然后点击`Run`：

![](../assets/6-17.png)

执行完成后，可以看到如下界面：

![](../assets/6-18.png)

也可以在airflow中看到执行情况：

![](../assets/5-3.png)

#### 查看采集的元数据信息

点开左侧菜单，可以看到采集的元数据信息：

![](../assets/6-19.png)

点开products表，可以看到采集的表信息：

![](../assets/6-20.png)

表的样例数据：

![](../assets/6-21.png)

表的剖析信息：

![](../assets/6-22.png)

列的剖析信息：

![](../assets/6-23.png)

如果是视图，还可以看到视图依赖的表信息，即数据血缘信息：

![](../assets/6-24.png)

通过上面的查询，查看存储过程：

![](../assets/6-25.png)

![](../assets/6-26.png)

到此，GaussDB的元数据信息已经采集完成。
