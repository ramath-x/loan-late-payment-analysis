# การวิเคราะห์การชำระเงินล่าช้าของเงินกู้

## คำอธิบายปัญหา

โปรเจกต์นี้ใช้ SQL เพื่อวิเคราะห์ข้อมูลการชำระเงินล่าช้าในระบบจัดการเงินกู้ โดยมีวัตถุประสงค์หลัก ดังนี้:

1. นับจำนวนครั้งที่แต่ละเงินกู้มีการชำระเงินล่าช้า
2. คำนวณจำนวนวันทั้งหมดที่แต่ละเงินกู้มีการชำระล่าช้า
3. หาจำนวนวันสูงสุดที่แต่ละเงินกู้เคยชำระล่าช้า

## วิธีการแก้ปัญหา

เราใช้ Common Table Expressions (CTEs) เพื่อประมวลผลข้อมูลเป็นขั้นตอน:

1. `late_payments`: ระบุการชำระเงินล่าช้าโดยเปรียบเทียบยอดค้างชำระปัจจุบันกับก่อนหน้า
2. `delinquency_periods`: กรองช่วงเวลาที่มีการชำระล่าช้าต่อเนื่องออก
3. `arrears_history`: คำนวณวันเริ่มต้นและสิ้นสุดของแต่ละช่วงการชำระล่าช้า

คำสั่ง SQL สุดท้ายจะรวมข้อมูลเหล่านี้เพื่อสร้างสถิติที่ต้องการ

## การตั้งค่าฐานข้อมูล

1. สร้างตาราง `transaction` ด้วยโครงสร้างดังนี้:

```sql
CREATE TABLE `transaction` (
    id INT AUTO_INCREMENT PRIMARY KEY,
    loan_id VARCHAR(255),
    effective_date TIMESTAMP, -- ใช้ TIMESTAMP สำหรับ MySQL
    arrears_amount DECIMAL(18,2)
);

```

2. insert ข้อมูล `transaction` ด้วยโครงสร้างดังนี้:

```sql

INSERT INTO `transaction` (loan_id, effective_date, arrears_amount) VALUES
('1001', '2024-01-01 00:00:00', 0),
('1001', '2024-01-02 00:00:00', 100),
('1001', '2024-01-02 00:00:00', 0),
('1002', '2024-01-01 00:00:00', 0),
('1002', '2024-01-02 00:00:00', 100),
('1002', '2024-01-05 00:00:00', 0),
('1003', '2024-01-01 00:00:00', 0),
('1003', '2024-01-02 00:00:00', 100),
('1003', '2024-01-05 00:00:00', 0),
('1003', '2024-02-02 00:00:00', 100),
('1003', '2024-02-04 00:00:00', 0),
('1004', '2024-01-01 00:00:00', 0),
('1004', '2024-01-02 00:00:00', 100),
('1004', '2024-01-05 00:00:00', -150),
('1004', '2024-02-02 00:00:00', -50),
('1004', '2024-02-04 00:00:00', 100),
('1004', '2024-02-04 00:00:00', 0),
('1005', '2024-01-01 00:00:00', 0),
('1005', '2024-01-02 00:00:00', 100),
('1005', '2024-01-05 00:00:00', 200),
('1005', '2024-01-07 00:00:00', 0),
('1005', '2024-01-08 00:00:00', 100),
('1005', '2024-01-09 00:00:00', 0);

```

3. สร้าง PROCEDURE ชื่อ get_arrears_history () ดังนี้:

```sql
DELIMITER //
CREATE PROCEDURE get_arrears_history () BEGIN
	WITH late_payments AS (
		SELECT
			*
		FROM
			(
			SELECT
				id,
				loan_id,
				effective_date,
				arrears_amount,
-- 				เอา arrears_amount ก่อนหน้าของ loan_id
				LAG ( arrears_amount ) OVER ( PARTITION BY loan_id ORDER BY id ) AS prev_arrears
			FROM
				`transaction`
			) t1
		WHERE
-- 		เอา เคสที่ แรกที่ยอดเป็น 0  ออก
			prev_arrears IS NOT NULL
-- 		ยอดติดลบหรือยอดเคงเหลือที่จ่ายออก ออก
			AND NOT ( arrears_amount < 0 AND prev_arrears < 0 )
		),
		delinquency_periods AS (
		SELECT
			*
		FROM
			late_payments
-- 			เอาแถวระหว่างที่ออกค้างชำระต่อเนื่องออก
		WHERE
		CASE
				WHEN NOT ( arrears_amount > 0 AND prev_arrears > 0 ) THEN
				'top_low' ELSE NULL
			END = 'top_low'
		)
		,
		arrears_history AS (
		SELECT
			*,
			effective_date AS start_date,
-- 			เพิ่มคอลัมน์เพื่อคำนวณระยะเวลาการค้างชำระ
			IFNULL( LEAD ( effective_date ) OVER ( PARTITION BY loan_id ORDER BY id ), CURRENT_DATE ) AS end_date,
			DATEDIFF( LEAD ( effective_date ) OVER ( PARTITION BY loan_id ORDER BY id ), effective_date ) AS date_diff
		FROM
			delinquency_periods
		)
		SELECT
		loan_id,
-- 		กรณีชำระตรงเวลา ไม่มีการค้างชำระ
	IF
		(
			sum( date_diff ) = 0,
			0,
		count( loan_id )) AS number_of_delinquency,
-- 		หาจำนวนวันทั้งหมด
		SUM(
		DATEDIFF( end_date, start_date )) AS total_day_of_delinquency,
-- 		หาจำนวนวันสูงสุดที่เคยค้างชำระ
		MAX(
		DATEDIFF( end_date, start_date )) AS maximum_day_of_delinquency
	FROM
		arrears_history
	WHERE
		( arrears_amount > 0 AND prev_arrears = 0 )
	GROUP BY
		loan_id ;
	END //
	DELIMITER;
```

## อธิบาย SQL
