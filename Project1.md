Yes. For your Multi-Branch Sales Management System, I recommend designing the MySQL database around Company → Branch → Users → Inventory → Sales → Invoices → Payments → Commission.

Below is a full production-oriented schema you can use as the database blueprint for your Spring Boot backend.

1. Overall Database Architecture

COMPANY
   │
   ├── BRANCHES
   │     │
   │     ├── BRANCH MANAGERS
   │     ├── SALESMEN
   │     ├── INVENTORY
   │     └── ORDERS / SALES
   │
   ├── PRODUCTS
   │     ├── CATEGORIES
   │     ├── BRANDS
   │     └── VARIANTS
   │
   ├── SHOPKEEPERS
   │     │
   │     └── ORDERS
   │
   ├── PRICING
   │
   ├── PAYMENTS
   │
   ├── RETURNS
   │
   ├── COMMISSION
   │
   └── REPORTING / AUDIT

I would use one MySQL database, not a separate database for every branch.


---

2. Main Roles

I recommend these roles:

SUPER_ADMIN
COMPANY_ADMIN
BRANCH_MANAGER
SALESMAN
SHOPKEEPER

You can later add:

ACCOUNTANT
DELIVERY_AGENT
INVENTORY_MANAGER

But don't make the first version unnecessarily complicated.


---

3. Complete Table List

Organization

companies
branches
users
roles
user_roles
branch_users

Product Management

categories
brands
products
product_variants
product_images
product_prices

Inventory

branch_inventory
stock_movements
stock_requests
stock_request_items
stock_transfers
stock_transfer_items

Customers / Shopkeepers

shopkeepers
shopkeeper_addresses
salesman_shopkeepers
customer_credit_accounts

Sales

orders
order_items
order_status_history
invoices
invoice_items

Payments

payments
payment_allocations

Returns

returns
return_items

Salesman Management

salesman_targets
commission_policies
commission_records
salary_records

Delivery

deliveries
delivery_items

Promotions

promotions
promotion_products

System

notifications
audit_logs


---

4. companies

This represents the main company.

CREATE TABLE companies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    name VARCHAR(150) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,

    email VARCHAR(150),
    phone VARCHAR(30),

    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100) DEFAULT 'India',
    postal_code VARCHAR(20),

    logo_url VARCHAR(500),

    tax_number VARCHAR(100),

    currency VARCHAR(10) DEFAULT 'INR',

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP
);


---

5. branches

Every branch belongs to a company.

CREATE TABLE branches (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    company_id BIGINT NOT NULL,

    name VARCHAR(150) NOT NULL,
    code VARCHAR(50) NOT NULL,

    email VARCHAR(150),
    phone VARCHAR(30),

    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100) DEFAULT 'India',
    postal_code VARCHAR(20),

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    CONSTRAINT fk_branch_company
        FOREIGN KEY (company_id)
        REFERENCES companies(id),

    UNIQUE(company_id, code)
);

Example:

Company
└── ABC Distribution

    ├── Kolkata Branch
    ├── Howrah Branch
    ├── Durgapur Branch
    └── Siliguri Branch


---

6. roles

CREATE TABLE roles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    name VARCHAR(50) NOT NULL UNIQUE,

    description VARCHAR(255)
);

Insert initial roles:

INSERT INTO roles (name, description) VALUES
('SUPER_ADMIN', 'System administrator'),
('COMPANY_ADMIN', 'Company administrator'),
('BRANCH_MANAGER', 'Branch manager'),
('SALESMAN', 'Salesman'),
('SHOPKEEPER', 'Shopkeeper');


---

7. users

Don't create separate login tables for manager, salesman and shopkeeper.

Use one authentication table.

CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(150) UNIQUE,

    password_hash VARCHAR(255) NOT NULL,

    full_name VARCHAR(150) NOT NULL,
    phone VARCHAR(30),

    profile_image_url VARCHAR(500),

    status ENUM(
        'ACTIVE',
        'INACTIVE',
        'SUSPENDED',
        'BLOCKED'
    ) DEFAULT 'ACTIVE',

    last_login_at DATETIME,

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP
);


---

8. user_roles

Many-to-many relationship.

CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,

    PRIMARY KEY (user_id, role_id),

    FOREIGN KEY (user_id)
        REFERENCES users(id)
        ON DELETE CASCADE,

    FOREIGN KEY (role_id)
        REFERENCES roles(id)
        ON DELETE CASCADE
);

Example:

Avijit
   ↓
SALESMAN


---

9. branch_users

This connects users to branches.

CREATE TABLE branch_users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    branch_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,

    joined_at DATE,

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (user_id)
        REFERENCES users(id),

    UNIQUE(branch_id, user_id)
);

This lets you determine:

User → Branch


---

10. Categories

CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    company_id BIGINT NOT NULL,

    name VARCHAR(100) NOT NULL,
    description TEXT,

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (company_id)
        REFERENCES companies(id),

    UNIQUE(company_id, name)
);


---

11. Brands

CREATE TABLE brands (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    company_id BIGINT NOT NULL,

    name VARCHAR(100) NOT NULL,
    description TEXT,

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (company_id)
        REFERENCES companies(id),

    UNIQUE(company_id, name)
);


---

12. Products

Product master.

CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    company_id BIGINT NOT NULL,
    category_id BIGINT,
    brand_id BIGINT,

    name VARCHAR(200) NOT NULL,

    sku VARCHAR(100) NOT NULL UNIQUE,

    description TEXT,

    unit_type ENUM(
        'PCS',
        'KG',
        'GRAM',
        'LITER',
        'ML',
        'BOX',
        'PACK',
        'DOZEN'
    ) DEFAULT 'PCS',

    tax_percent DECIMAL(5,2) DEFAULT 0,

    status ENUM(
        'ACTIVE',
        'INACTIVE',
        'DISCONTINUED'
    ) DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (company_id)
        REFERENCES companies(id),

    FOREIGN KEY (category_id)
        REFERENCES categories(id),

    FOREIGN KEY (brand_id)
        REFERENCES brands(id)
);


---

13. Product Variants

For example:

Coca Cola
 ├── 250ml
 ├── 500ml
 ├── 1L
 └── 2L

CREATE TABLE product_variants (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    product_id BIGINT NOT NULL,

    variant_name VARCHAR(150) NOT NULL,

    sku VARCHAR(100) NOT NULL UNIQUE,

    barcode VARCHAR(100) UNIQUE,

    unit_value DECIMAL(12,3),

    unit_type VARCHAR(30),

    manufacturing_cost DECIMAL(12,2) NOT NULL DEFAULT 0,

    mrp DECIMAL(12,2) NOT NULL DEFAULT 0,

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (product_id)
        REFERENCES products(id)
);


---

14. Product Images

CREATE TABLE product_images (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    product_id BIGINT NOT NULL,

    image_url VARCHAR(500) NOT NULL,

    is_primary BOOLEAN DEFAULT FALSE,

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (product_id)
        REFERENCES products(id)
        ON DELETE CASCADE
);


---

15. Product Pricing

This is very important for your business model.

CREATE TABLE product_prices (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    product_variant_id BIGINT NOT NULL,

    price_type ENUM(
        'COMPANY_TO_BRANCH',
        'BRANCH_TO_SHOPKEEPER',
        'MRP'
    ) NOT NULL,

    price DECIMAL(12,2) NOT NULL,

    effective_from DATETIME NOT NULL,
    effective_to DATETIME,

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    FOREIGN KEY (product_variant_id)
        REFERENCES product_variants(id)
);

Example:

Product: Biscuit 500g

Manufacturing Cost = ₹30

COMPANY_TO_BRANCH = ₹40

BRANCH_TO_SHOPKEEPER = ₹60

MRP = ₹100


---

16. Branch Inventory

This stores the current stock.

CREATE TABLE branch_inventory (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    branch_id BIGINT NOT NULL,
    product_variant_id BIGINT NOT NULL,

    quantity DECIMAL(12,3) NOT NULL DEFAULT 0,

    reserved_quantity DECIMAL(12,3)
        NOT NULL DEFAULT 0,

    minimum_stock DECIMAL(12,3)
        NOT NULL DEFAULT 0,

    maximum_stock DECIMAL(12,3),

    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (product_variant_id)
        REFERENCES product_variants(id),

    UNIQUE(branch_id, product_variant_id)
);

Available stock:

available_stock =
quantity - reserved_quantity


---

17. Stock Movements

This is one of the most important tables.

CREATE TABLE stock_movements (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    branch_id BIGINT NOT NULL,
    product_variant_id BIGINT NOT NULL,

    movement_type ENUM(
        'PURCHASE',
        'SALE',
        'RETURN_IN',
        'RETURN_OUT',
        'TRANSFER_IN',
        'TRANSFER_OUT',
        'ADJUSTMENT',
        'DAMAGE',
        'EXPIRY'
    ) NOT NULL,

    quantity DECIMAL(12,3) NOT NULL,

    reference_type VARCHAR(50),
    reference_id BIGINT,

    remarks VARCHAR(500),

    created_by BIGINT,

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (product_variant_id)
        REFERENCES product_variants(id),

    FOREIGN KEY (created_by)
        REFERENCES users(id)
);

This gives you an audit trail.

Example:

+100 PURCHASE
-20 SALE
+5 RETURN_IN
-3 DAMAGE

Current stock = 82


---

18. Branch Stock Requests

When a branch needs products from the company.

CREATE TABLE stock_requests (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    branch_id BIGINT NOT NULL,

    request_number VARCHAR(50) NOT NULL UNIQUE,

    status ENUM(
        'PENDING',
        'APPROVED',
        'REJECTED',
        'PARTIALLY_FULFILLED',
        'FULFILLED',
        'CANCELLED'
    ) DEFAULT 'PENDING',

    requested_by BIGINT NOT NULL,
    approved_by BIGINT,

    remarks TEXT,

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (requested_by)
        REFERENCES users(id),

    FOREIGN KEY (approved_by)
        REFERENCES users(id)
);

Items:

CREATE TABLE stock_request_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    stock_request_id BIGINT NOT NULL,

    product_variant_id BIGINT NOT NULL,

    requested_quantity DECIMAL(12,3) NOT NULL,

    approved_quantity DECIMAL(12,3),

    FOREIGN KEY (stock_request_id)
        REFERENCES stock_requests(id)
        ON DELETE CASCADE,

    FOREIGN KEY (product_variant_id)
        REFERENCES product_variants(id)
);


---

19. Stock Transfers Between Branches

CREATE TABLE stock_transfers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    transfer_number VARCHAR(50) NOT NULL UNIQUE,

    from_branch_id BIGINT NOT NULL,
    to_branch_id BIGINT NOT NULL,

    status ENUM(
        'PENDING',
        'APPROVED',
        'DISPATCHED',
        'RECEIVED',
        'CANCELLED'
    ) DEFAULT 'PENDING',

    created_by BIGINT NOT NULL,
    approved_by BIGINT,

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (from_branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (to_branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (created_by)
        REFERENCES users(id),

    FOREIGN KEY (approved_by)
        REFERENCES users(id)
);

Items:

CREATE TABLE stock_transfer_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    stock_transfer_id BIGINT NOT NULL,

    product_variant_id BIGINT NOT NULL,

    quantity DECIMAL(12,3) NOT NULL,

    received_quantity DECIMAL(12,3),

    FOREIGN KEY (stock_transfer_id)
        REFERENCES stock_transfers(id)
        ON DELETE CASCADE,

    FOREIGN KEY (product_variant_id)
        REFERENCES product_variants(id)
);


---

20. Shopkeepers

CREATE TABLE shopkeepers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    user_id BIGINT NOT NULL,

    company_id BIGINT NOT NULL,

    shop_name VARCHAR(200) NOT NULL,

    business_registration_number VARCHAR(100),

    tax_number VARCHAR(100),

    credit_limit DECIMAL(12,2) DEFAULT 0,

    payment_terms_days INT DEFAULT 0,

    status ENUM(
        'ACTIVE',
        'INACTIVE',
        'BLOCKED'
    ) DEFAULT 'ACTIVE',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id)
        REFERENCES users(id),

    FOREIGN KEY (company_id)
        REFERENCES companies(id)
);


---

21. Shopkeeper Addresses

CREATE TABLE shopkeeper_addresses (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    shopkeeper_id BIGINT NOT NULL,

    address_type ENUM(
        'SHOP',
        'BILLING',
        'DELIVERY'
    ) DEFAULT 'SHOP',

    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),

    city VARCHAR(100),
    state VARCHAR(100),
    postal_code VARCHAR(20),

    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),

    is_default BOOLEAN DEFAULT FALSE,

    FOREIGN KEY (shopkeeper_id)
        REFERENCES shopkeepers(id)
        ON DELETE CASCADE
);


---

22. Salesman-Shopkeeper Assignment

CREATE TABLE salesman_shopkeepers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    salesman_id BIGINT NOT NULL,
    shopkeeper_id BIGINT NOT NULL,

    assigned_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    status ENUM('ACTIVE', 'INACTIVE')
        DEFAULT 'ACTIVE',

    FOREIGN KEY (salesman_id)
        REFERENCES users(id),

    FOREIGN KEY (shopkeeper_id)
        REFERENCES shopkeepers(id),

    UNIQUE(salesman_id, shopkeeper_id)
);


---

23. Customer Credit Account

CREATE TABLE customer_credit_accounts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    shopkeeper_id BIGINT NOT NULL UNIQUE,

    credit_limit DECIMAL(12,2) DEFAULT 0,

    current_outstanding DECIMAL(12,2)
        DEFAULT 0,

    status ENUM(
        'ACTIVE',
        'SUSPENDED'
    ) DEFAULT 'ACTIVE',

    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (shopkeeper_id)
        REFERENCES shopkeepers(id)
);


---

24. Orders

This is your main sales transaction.

CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    order_number VARCHAR(50) NOT NULL UNIQUE,

    company_id BIGINT NOT NULL,
    branch_id BIGINT NOT NULL,

    shopkeeper_id BIGINT NOT NULL,

    salesman_id BIGINT,

    order_source ENUM(
        'SALESMAN',
        'SHOPKEEPER',
        'ADMIN'
    ) DEFAULT 'SALESMAN',

    status ENUM(
        'PENDING',
        'CONFIRMED',
        'PROCESSING',
        'PACKED',
        'DISPATCHED',
        'DELIVERED',
        'PARTIALLY_DELIVERED',
        'CANCELLED',
        'REJECTED'
    ) DEFAULT 'PENDING',

    subtotal DECIMAL(12,2) DEFAULT 0,

    discount_amount DECIMAL(12,2) DEFAULT 0,

    tax_amount DECIMAL(12,2) DEFAULT 0,

    total_amount DECIMAL(12,2) DEFAULT 0,

    notes TEXT,

    created_by BIGINT,

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (company_id)
        REFERENCES companies(id),

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (shopkeeper_id)
        REFERENCES shopkeepers(id),

    FOREIGN KEY (salesman_id)
        REFERENCES users(id),

    FOREIGN KEY (created_by)
        REFERENCES users(id)
);


---

25. Order Items

CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    order_id BIGINT NOT NULL,

    product_variant_id BIGINT NOT NULL,

    quantity DECIMAL(12,3) NOT NULL,

    unit_price DECIMAL(12,2) NOT NULL,

    discount_amount DECIMAL(12,2) DEFAULT 0,

    tax_percent DECIMAL(5,2) DEFAULT 0,

    tax_amount DECIMAL(12,2) DEFAULT 0,

    line_total DECIMAL(12,2) NOT NULL,

    FOREIGN KEY (order_id)
        REFERENCES orders(id)
        ON DELETE CASCADE,

    FOREIGN KEY (product_variant_id)
        REFERENCES product_variants(id)
);

Important: unit_price is stored here even though the product has a price table.

Why?

Because tomorrow the price can change from ₹60 to ₹65.

Old invoices must still show ₹60.


---

26. Order Status History

CREATE TABLE order_status_history (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    order_id BIGINT NOT NULL,

    old_status VARCHAR(50),
    new_status VARCHAR(50) NOT NULL,

    changed_by BIGINT,

    remarks VARCHAR(500),

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (order_id)
        REFERENCES orders(id)
        ON DELETE CASCADE,

    FOREIGN KEY (changed_by)
        REFERENCES users(id)
);

This allows:

PENDING
   ↓
CONFIRMED
   ↓
PACKED
   ↓
DISPATCHED
   ↓
DELIVERED


---

27. Invoices

CREATE TABLE invoices (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    invoice_number VARCHAR(50) NOT NULL UNIQUE,

    order_id BIGINT NOT NULL UNIQUE,

    branch_id BIGINT NOT NULL,

    shopkeeper_id BIGINT NOT NULL,

    invoice_date DATETIME DEFAULT CURRENT_TIMESTAMP,

    due_date DATE,

    subtotal DECIMAL(12,2) DEFAULT 0,

    discount_amount DECIMAL(12,2) DEFAULT 0,

    tax_amount DECIMAL(12,2) DEFAULT 0,

    total_amount DECIMAL(12,2) DEFAULT 0,

    paid_amount DECIMAL(12,2) DEFAULT 0,

    outstanding_amount DECIMAL(12,2) DEFAULT 0,

    status ENUM(
        'UNPAID',
        'PARTIALLY_PAID',
        'PAID',
        'OVERDUE',
        'CANCELLED'
    ) DEFAULT 'UNPAID',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (order_id)
        REFERENCES orders(id),

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    FOREIGN KEY (shopkeeper_id)
        REFERENCES shopkeepers(id)
);


---

28. Invoice Items

CREATE TABLE invoice_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    invoice_id BIGINT NOT NULL,

    product_variant_id BIGINT NOT NULL,

    product_name VARCHAR(200) NOT NULL,

    sku VARCHAR(100),

    quantity DECIMAL(12,3) NOT NULL,

    unit_price DECIMAL(12,2) NOT NULL,

    discount_amount DECIMAL(12,2) DEFAULT 0,

    tax_percent DECIMAL(5,2) DEFAULT 0,

    tax_amount DECIMAL(12,2) DEFAULT 0,

    line_total DECIMAL(12,2) NOT NULL,

    FOREIGN KEY (invoice_id)
        REFERENCES invoices(id)
        ON DELETE CASCADE
);


---

29. Payments

CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    payment_number VARCHAR(50) NOT NULL UNIQUE,

    shopkeeper_id BIGINT NOT NULL,

    branch_id BIGINT NOT NULL,

    amount DECIMAL(12,2) NOT NULL,

    payment_method ENUM(
        'CASH',
        'UPI',
        'BANK_TRANSFER',
        'CARD',
        'CHEQUE',
        'ONLINE'
    ) NOT NULL,

    transaction_reference VARCHAR(150),

    payment_date DATETIME DEFAULT CURRENT_TIMESTAMP,

    status ENUM(
        'PENDING',
        'SUCCESS',
        'FAILED',
        'REFUNDED'
    ) DEFAULT 'SUCCESS',

    received_by BIGINT,

    notes VARCHAR(500),

    FOREIGN KEY (shopkeeper_id)
        REFERENCES shopkeepers(id),

    FOREIGN KEY (branch_id)
        REFERENCES branches(id),

    
