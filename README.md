# AWS-RECEIPTS-PART-2-IAC
After we had a working MVP for the site, I sat with the customer to walk through what we built. He paused, looked at me and said:

> “Wait, you’re telling me you clicked through all those AWS screens manually? Why didn’t you use Infrastructure as Code?”
> 

Ouch. He was right. While “clickops” got us fast results, even a shortcut deserves a better shortcut. Welcome to **CloudFormation**.

---

## 📦 Step 1: What is CloudFormation?

**CloudFormation** is AWS’s native tool for **Infrastructure as Code (IaC)**. It allows you to:

- Write a **template** (YAML or JSON)
- Launch a **stack** (an instance of the infrastructure)
- Manage and update your resources declaratively

Think of it like this:

- **Template** = your blueprint (YAML file)
- **Stack** = your running version of that template
- **StackSet** = deploying the same stack to multiple accounts or regions

---

## 🔍 Step 2: Why IaC Over ClickOps?

Manual setup is fine for exploration — but not for reproducibility or scaling. Here's why we shifted:

ClickOps:

- ❌ Manual
- ❌ Hard to version
- ❌ Easy to forget steps

IaC with CloudFormation:

- ✅ Repeatable and version-controlled
- ✅ Can be peer-reviewed and shared
- ✅ Great for automation and pipelines

Here is the Diagram I prepared for the implenatation using IAC (CloudFormation)

![image.png](attachment:10f9cb01-07ae-4eb8-b95e-8bacb1ffc98a:image.png)

---

## 🔐 Step 3: Prepare a Key Pair

Before launching the stack:

1. Go to [EC2 > Key Pairs](https://console.aws.amazon.com/ec2/v2/home?#KeyPairs)
2. Click **Create Key Pair**
3. Name it `ReceiptsEC2`
4. Download the `.pem` file to your computer (store it safely)

We'll reference this key in the CloudFormation template to allow SSH access.

---

## 🧱 Step 4: YAML Structure Overview

Let’s break the template into clear blocks:

### A. Metadata

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: EC2 Instance with Flask App and MySQL using UserData (Free Tier)

```

### B. Security Group

```yaml
Resources:
  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP from anywhere
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          CidrIp: 0.0.0.0/0

```

### C. EC2 Instance Definition

```yaml
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0c02fb55956c7d316
      KeyName: ReceiptsEC2
      SecurityGroupIds:
        - !Ref EC2SecurityGroup
      Tags:
        - Key: Name
          Value: IAC-Receipts-EC2

```

### D. User Data (Boot Script)

Embedded into the EC2 block as `UserData`, this script:

- Installs Python, Flask, and MySQL
- Initializes the database and table
- Uploads the Flask app and HTML
- Launches the app on port 8080

```yaml
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y python3 git mysql-server
          pip3 install flask pymysql

          systemctl enable mysqld
          systemctl start mysqld

          sleep 10

          mysql -u root <<EOF
          CREATE DATABASE IF NOT EXISTS recipes_db;
          USE recipes_db;
          CREATE TABLE IF NOT EXISTS recipe_images (
            id INT AUTO_INCREMENT PRIMARY KEY,
            filename VARCHAR(255) NOT NULL,
            title VARCHAR(255),
            upload_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
          );
          EOF

          mkdir -p /home/ec2-user/recipes-site/static/uploads
          mkdir -p /home/ec2-user/recipes-site/templates

          cat <<EOF > /home/ec2-user/recipes-site/app.py
          from flask import Flask, request, render_template, redirect, url_for
          import os
          from werkzeug.utils import secure_filename

          app = Flask(__name__)
          app.config['UPLOAD_FOLDER'] = 'static/uploads'
          app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024
          ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif'}

          def allowed_file(filename):
              return '.' in filename and \
                     filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

          @app.route('/', methods=['GET', 'POST'])
          def index():
              if request.method == 'POST':
                  if 'file' not in request.files:
                      return 'No file part'
                  file = request.files['file']
                  if file.filename == '':
                      return 'No selected file'
                  if file and allowed_file(file.filename):
                      filename = secure_filename(file.filename)
                      filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
                      file.save(filepath)
                      return redirect(url_for('uploaded_file', filename=filename))
              return render_template('index.html')

          @app.route('/uploads/<filename>')
          def uploaded_file(filename):
              return f"""
              <h2>Upload successful!</h2>
              <img src='/static/uploads/{filename}' width='300'>
              <br><br><a href="/">Upload another</a>
              """

          if __name__ == '__main__':
              app.run(host='0.0.0.0', port=8080)
          EOF

          cat <<EOF > /home/ec2-user/recipes-site/templates/index.html
          <!doctype html>
          <html>
          <head><title>Upload Recipe Image</title></head>
          <body>
            <h1>Upload a Recipe Photo</h1>
            <form method=post enctype=multipart/form-data>
              <input type=file name=file>
              <input type=submit value=Upload>
            </form>
          </body>
          </html>
          EOF

          cd /home/ec2-user/recipes-site
          FLASK_APP=app.py nohup flask run --host=0.0.0.0 --port=8080 &

```

---

## 🚀 Step 5: Deploying the Stack

1. Go to [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
2. Click **Create Stack > With new resources**
3. Upload your YAML file
4. Click through the wizard and launch

Once the stack is ready, go to EC2 and grab the public IP:

```bash
http://<your-ec2-public-ip>:8080

```

---

Now the Internet site is operational ! 

![image.png](attachment:86777202-3f7b-45cd-9323-e2f70cdf4590:image.png)

## ✅ Summary

We’ve now evolved from a one-off click-based setup to a fully repeatable, automated deployment. Here’s what we accomplished:

### 🔹 Highlights:

- Created a Flask and MySQL app stack from YAML
- Embedded database creation and app deployment in one step
- Opened HTTP and SSH ports to the world
- No manual configuration after launch

### 🔗 References:

- [AWS CloudFormation Overview](https://aws.amazon.com/cloudformation/)
- [UserData Reference](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [Amazon Linux AMI Info](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-linux-ami-basics.html)
- [Flask Quickstart](https://flask.palletsprojects.com/en/latest/quickstart/)
- [MySQL Docs](https://dev.mysql.com/doc/)
- [AWS Key Pairs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)