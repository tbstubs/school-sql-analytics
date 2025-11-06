# school-sql-analytics
This repository showcases a database schema that resembles a school's Student Information System. It is popluated with data for students enrolled in a technical school in eastern Massachusetts. 

To access the database, simply download or pull the file 'Test_School_DB - With Test Data.sql' and load it into your PostgreSQL application of choice. Be sure to have PostgreSQL installed on your machine from the official website. 

Once the file is imported properly, you can execute the following example commands to query data within the database (Query CODE | Commented Description):

1. --Shows all the columns and rows within the students table
   SELECT * FROM students; 
  
2. --Shows all te columns and rows within the enrollments table
   SELECT * FROM enrollments; 
  
3. --Shows all the studnets in 9th grade or higher
   SELECT * FROM students
   WHERE grade >= 9;
   
4. --Shows all distinct cities the students live in
   SELECT DISTINCT city FROM students;

5. --Shows how many students live in each respective city
   SELECT city, COUNT(city) FROM students
   GROUP BY city;

6. --Shows all the invoices that have payments made over $1,100.00 and any related information
   SELECT * FROM invoices
   INNER JOIN payments ON invoices.invoice_id = payments.fk_invoice_id
   WHERE payment_amount::numeric::integer > 1100
   ORDER BY payer_first_name ASC;
   
7. --Shows all the students enrolled in 'Computer Science'
   SELECT students.first_name, students.last_name, enrollments.enrollment_name FROM students
   JOIN students_enrollments ON students.student_id = students_enrollments.fk_student_id
   JOIN enrollments ON students_enrollments.fk_enrollment_id = enrollments.enrollment_id
   WHERE enrollment_name = 'Computer Science'
   ORDER BY last_name;
    
