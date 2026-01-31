
# Three-Tier PHP Registration Form — Docker Deployment 

## Overview

The __Three-Tier PHP Registration Form — Docker Deployment__ project demonstrates a fully containerized, production-style web application using a 3-tier architecture. It separates the application into __Presentation (Frontend), Application (PHP Backend), and Database (MySQL)__ tiers — each running in isolated Docker containers for maximum scalability, portability, and modularity.

![](./img/three-tier-architecture-diagram.png)

Three tiers used
* __Web (Reverse proxy / static files)__ — Nginx serving static assets and forwarding PHP requests to PHP-FPM.

* __Application (PHP-FPM)__ — PHP runtime that executes the registration form logic.

* __Database (MySQL)__ — Relational database that stores registered user data.

This README explains folder structure, how to build and run locally with Docker, how the services communicate, and production considerations.

## Step-by-Step Setup
### Step 1: Launch EC2 Instance
Launch an Amazon Linux EC2 instance.
Connect to it via SSH:

```bash
ssh -i <YOUR_KEY_PEM>.pem ec2-user@<EC2-Public-IP>
```
### Step 2: Install Docker & Docker Compose
```bash
sudo yum update -y
sudo amazon-linux-extras install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```
__Install Docker Compose:__
```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/d
ocker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compos
e
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
Logout and login again to apply the group changes.

### Step 3: Create Project Structure
```bash 
mkdir -p /home/ec2-user/threetier/{web,app,db}
cd /home/ec2-user/threetier
mkdir -p web/{code,config}
mkdir -p app/code
```
###  Folder structure

![](./img/tree.png)

### Step 4: Prepare Web Tier (Nginx)
(a) Get the default Nginx configuration

Create a temporary container to copy the default config:
```bash 
docker run -d --name temp-nginx nginx
docker exec -it temp-nginx cat /etc/nginx/conf.d/default.conf > web/config/d
efault.conf
docker rm -f temp-nginx
```
__(b) Edit the default.conf file
Open and edit:__
```bash
vim web/config/default.conf
```

Replace the PHP block with this configuration:
```bash 
location ~ \.php$ {
 root /app;
 fastcgi_pass app:9000;
 fastcgi_index index.php;
 fastcgi_param SCRIPT_FILENAME /app/$fastcgi_script_name;
 include fastcgi_params;
}
```
![](./img/configurationfile.png)

### Step 5: Create Frontend (Signup Form)
File: ```/web/code/signup.html```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Signup Form</title>

    <style>
        body {
            font-family: "Poppins", Arial, sans-serif;
            background: linear-gradient(135deg, #fbc2eb, #a6c1ee);
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }

        .form-container {
            background: rgba(255, 255, 255, 0.95);
            padding: 30px;
            width: 360px;
            border-radius: 16px;
            box-shadow: 0 20px 40px rgba(0, 0, 0, 0.15);
            backdrop-filter: blur(8px);
        }

        h2 {
            text-align: center;
            color: #4b4b4b;
            margin-bottom: 25px;
            letter-spacing: 1px;
        }

        label {
            font-weight: 500;
            color: #555;
            font-size: 14px;
        }

        input[type="text"],
        input[type="email"],
        input[type="url"],
        textarea {
            width: 100%;
            padding: 10px 12px;
            margin-top: 6px;
            border-radius: 8px;
            border: 1px solid #ddd;
            box-sizing: border-box;
            font-size: 14px;
            transition: all 0.3s ease;
        }

        input:focus,
        textarea:focus {
            outline: none;
            border-color: #a18cd1;
            box-shadow: 0 0 8px rgba(161, 140, 209, 0.4);
        }

        textarea {
            resize: none;
        }

        .gender-group {
            margin-top: 8px;
            color: #555;
            font-size: 14px;
        }

        input[type="radio"] {
            accent-color: #a18cd1;
            margin-right: 6px;
        }

        input[type="submit"] {
            width: 100%;
            padding: 12px;
            background: linear-gradient(135deg, #a18cd1, #fbc2eb);
            border: none;
            color: white;
            font-size: 16px;
            border-radius: 10px;
            cursor: pointer;
            margin-top: 20px;
            transition: transform 0.2s ease, box-shadow 0.2s ease;
        }

        input[type="submit"]:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 20px rgba(0, 0, 0, 0.2);
        }
    </style>
</head>

<body>

    <div class="form-container">
        <h2>Signup Form</h2>

        <form action="submit.php" method="post">
            <label>Name</label>
            <input type="text" name="name" required>

            <br><br>

            <label>Email</label>
            <input type="email" name="email" required>

            <br><br>

            <label>Website</label>
            <input type="url" name="website">

            <br><br>

            <label>Comment</label>
            <textarea name="comment" rows="4"></textarea>

            <br><br>

            <label>Gender</label>
            <div class="gender-group">
                <input type="radio" name="gender" value="female" required> Female<br>
                <input type="radio" name="gender" value="male"> Male<br>
                <input type="radio" name="gender" value="other"> Other
            </div>

            <input type="submit" value="Submit">
        </form>
    </div>

</body>
</html>



```

### Step 6: Create Application Tier (PHP-FPM)
File: ```/app/code/submit.php```

```php
<?php
error_reporting(E_ALL);
ini_set('display_errors', 1);

$name = $_POST['name'];
$email = $_POST['email'];
$website = $_POST['website'];
$comment = $_POST['comment'];
$gender = $_POST['gender'];

$servername = "db";
$username = "root";
$password = "root";
$dbname = "FCT";

$conn = mysqli_connect($servername, $username, $password, $dbname);
if (!$conn) {
    die("Connection failed: " . mysqli_connect_error());
}

$sql = "INSERT INTO users (name, email, website, comment, gender)
        VALUES ('$name', '$email', '$website', '$comment', '$gender')";
?>

<!DOCTYPE html>
<html>
<head>
    <title>Form Submission Result</title>

    <style>
        body {
            font-family: Arial, Helvetica, sans-serif;
            background: linear-gradient(135deg, #667eea, #764ba2);
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }

        .result-box {
            background-color: #ffffff;
            padding: 25px 30px;
            width: 420px;
            border-radius: 10px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.25);
        }

        h2 {
            text-align: center;
            color: #2e7d32;
        }

        h3 {
            color: #333;
            margin-top: 20px;
        }

        ul {
            list-style: none;
            padding: 0;
        }

        ul li {
            background-color: #f4f6f8;
            margin-bottom: 10px;
            padding: 10px;
            border-radius: 5px;
            color: #444;
        }

        ul li strong {
            color: #222;
        }

        .error {
            color: #c62828;
            text-align: center;
            font-weight: bold;
        }
    </style>
</head>

<body>

<div class="result-box">
<?php
if (mysqli_query($conn, $sql)) {
    echo "<h2>New record created successfully!</h2>";
    echo "<h3>Submitted Information:</h3>";
    echo "<ul>";
    echo "<li><strong>Name:</strong> " . htmlspecialchars($name) . "</li>";
    echo "<li><strong>Email:</strong> " . htmlspecialchars($email) . "</li>";
    echo "<li><strong>Website:</strong> " . htmlspecialchars($website) . "</li>";
    echo "<li><strong>Comment:</strong> " . htmlspecialchars($comment) . "</li>";
    echo "<li><strong>Gender:</strong> " . htmlspecialchars($gender) . "</li>";
    echo "</ul>";
} else {
    echo "<p class='error'>Error: " . mysqli_error($conn) . "</p>";
}

mysqli_close($conn);
?>
</div>

</body>
</html>

```

### Step 7: Create Database Tier
__File:__ ```/db/Dockerfile```

```docker
FROM mysql
ENV MYSQL_ROOT_PASSWORD=root
ENV MYSQL_DATABASE=studentapp
COPY init.sql /docker-enterypoint-initdb.d/
EXPOSE 3306
CMD ["mysqld"]
```

__File:__ ```/db/init.sql```
```sql
CREATE TABLE users (
 id INT PRIMARY KEY AUTO_INCREMENT,
 name VARCHAR(20),
 email VARCHAR(100),
 website VARCHAR(255),
 gender VARCHAR(6),
 comment VARCHAR(100)
);
```
### Step 8: Create Docker Compose File
__File:__ /docker-compose.yml

```yml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./web/code/:/usr/share/nginx/html/
      - ./web/config/:/etc/nginx/conf.d/
    networks:
      - webnet
    depends_on:
      - app
      - db

  app:
    image: bitnami/php-fpm
    volumes:
      - ./app/code/:/app/
    networks:
      - webnet
      - dbnet
    depends_on:
      - db

  db:
    build: ./db/
    volumes:
      - myvolume:/var/lib/mysql
    networks:
      - dbnet

networks:
  webnet:
  dbnet:

volumes:
  myvolume:

```

### Step 9: Start the Containers
```bash
docker-compose up -d
```
![](./img/composefile-run.png)

Check running containers:
```bash
docker ps
```
![](./img/containers.png)

### Step 10: Verify Application
Open your EC2 Public IP in a browser:
```bash
 http://<EC2-Public-IP>/signup.html
```
Fill out the form and click Submit.
You should see a confirmation message with the entered data.

### Output
![](./img/form-UI.png)
![](./img/form-submission-UI.png)

### Step 11: Verify Database
To confirm that the data is saved in MySQL:
```
docker exec -it <db_container_id> mysql -u root -p root
use FCT; 
FCT mysql > SELECT * FROM users;
```
![](./img/database-login.png)

![](./img/stored-data.png)


### Conclusion
The __Three-Tier PHP Registration Form — Docker Deployment__ project provides a practical and efficient demonstration of how modern applications can be architected, containerized, and deployed using Docker. By separating the frontend, backend, and database into dedicated containers, the setup ensures better scalability, reliability, and maintainability. Docker Compose further simplifies orchestration, making deployments consistent across different environments. Overall, this project not only showcases a clean three-tier architecture but also serves as a strong foundation for learners and developers looking to understand real-world container-based application deployment.
