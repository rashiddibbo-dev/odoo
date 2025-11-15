# Deploying Odoo to Google Cloud Platform (GCP)

This guide provides a step-by-step process for deploying an Odoo application to Google Cloud Platform. It assumes you have a Google Cloud account with billing enabled.

## 1. Set up a Google Cloud Project

Every resource in GCP belongs to a project.

1.  Go to the [Google Cloud Console](https://console.cloud.google.com/).
2.  Click the project drop-down in the top-left corner and click **New Project**.
3.  Give your project a name (e.g., `odoo-deployment`) and click **Create**.
4.  Make sure your new project is selected in the project drop-down.

## 2. Create a Cloud SQL for PostgreSQL Instance

Odoo needs a PostgreSQL database. Cloud SQL is a fully-managed database service that makes it easy to set up, maintain, manage, and administer your relational databases on Google Cloud Platform.

1.  In the Cloud Console, navigate to **SQL** from the left-hand menu (under "Databases").
2.  Click **Create Instance**.
3.  Choose **PostgreSQL**.
4.  Provide an **Instance ID** (e.g., `odoo-db`).
5.  Set a strong **password** for the `postgres` user. Remember this password, as you'll need it later.
6.  For **Database version**, choose a version compatible with this Odoo project (e.g., PostgreSQL 14 or higher).
7.  Choose a **Region** and **Zone** that is geographically close to you.
8.  Under **Connections**, check **Public IP**. This allows your Compute Engine instance to connect to the database. For better security in a production environment, you would use a private IP and a VPC network.
9.  Click **Create Instance**. It will take a few minutes for the instance to be ready.

Once the instance is created:
1.  Click on the instance name to open its overview.
2.  Go to the **Databases** tab and click **Create database**. Name it `odoo`.
3.  Go to the **Users** tab and click **Create user account**. Create a user named `odoo` and set a secure password.

## 3. Create a Compute Engine VM

This virtual machine will run the Odoo application.

1.  In the Cloud Console, navigate to **Compute Engine** > **VM instances**.
2.  Click **Create Instance**.
3.  Give the instance a **Name** (e.g., `odoo-server`).
4.  Choose the same **Region** and **Zone** as your Cloud SQL instance.
5.  For **Machine type**, a general-purpose machine like `e2-medium` (2 vCPUs, 4 GB memory) is a good starting point.
6.  For the **Boot disk**, click **Change**. Select **Ubuntu** and a version like **22.04 LTS**. Click **Select**.
7.  Under **Firewall**, check **Allow HTTP traffic** and **Allow HTTPS traffic**. This will create firewall rules to allow web traffic to your VM.
8.  Click **Create**.

## 4. Configure the VM and Install Dependencies

1.  Once the VM is created, click the **SSH** button next to it to open a terminal session in your browser.
2.  **Update the package list:**
    ```bash
    sudo apt-get update && sudo apt-get upgrade -y
    ```
3.  **Install PostgreSQL client and other dependencies:**
    ```bash
    sudo apt-get install -y python3-pip python3-dev build-essential libxslt-dev libzip-dev libldap2-dev libsasl2-dev libpq-dev git wkhtmltopdf
    ```
4.  **Install Python dependencies for Odoo:**
    ```bash
    pip3 install -r requirements.txt
    pip3 install phonenumbers
    ```
    *Note: You will first need to get your application code onto the VM for `requirements.txt` to be available.*

## 5. Get the Application Code onto the VM

You can get your code onto the VM in several ways. Using `git` is the most common.

1.  In the SSH terminal on your VM, clone your git repository:
    ```bash
    git clone <your-repository-url>
    cd <your-repository-directory>
    ```
2.  Now, you can run the `pip3 install -r requirements.txt` command from the previous step.

## 6. Configure Odoo to Connect to Cloud SQL

You need to tell Odoo where to find its database.

1.  Find the **Public IP address** of your Cloud SQL instance from its overview page in the Google Cloud Console.
2.  Edit the Odoo configuration file (`odoo.conf` or create one). It's good practice not to use command-line arguments for production.
    ```ini
    [options]
    addons_path = /path/to/your/odoo/addons,/path/to/your/custom/addons
    admin_passwd = <your_admin_password>
    db_host = <your_cloud_sql_public_ip>
    db_port = 5432
    db_user = odoo
    db_password = <your_odoo_db_user_password>
    db_name = odoo
    http_port = 8069
    ```
3.  **Authorize the connection from your VM to Cloud SQL:**
    *   Find the **External IP** of your Compute Engine VM.
    *   In the Cloud SQL instance details, go to the **Connections** tab.
    *   Under **Authorized networks**, click **Add Network**.
    *   Enter a name (e.g., `odoo-vm`) and the external IP of your VM. Click **Done** and then **Save**.

## 7. Running Odoo as a Service

Running Odoo directly in the terminal is not ideal for a production server. You should run it as a systemd service.

1.  Create a service file:
    ```bash
    sudo nano /etc/systemd/system/odoo.service
    ```
2.  Add the following content, adjusting paths as necessary:
    ```ini
    [Unit]
    Description=Odoo
    Requires=postgresql.service
    After=network.target postgresql.service

    [Service]
    Type=simple
    SyslogIdentifier=odoo
    PermissionsStartOnly=true
    User=<your_vm_user>
    Group=<your_vm_user>
    ExecStart=/usr/bin/python3 /path/to/your/odoo-bin -c /path/to/your/odoo.conf
    StandardOutput=journal+console

    [Install]
    WantedBy=multi-user.target
    ```
3.  **Enable and start the service:**
    ```bash
    sudo systemctl enable odoo.service
    sudo systemctl start odoo.service
    ```
4.  **Check the status:**
    ```bash
    sudo systemctl status odoo.service
    ```

## 8. Configuring Firewall Rules

Although you allowed HTTP/HTTPS traffic when creating the VM, you may need to explicitly open port 8069 if you plan to access it directly. For a production setup, you would typically use a reverse proxy like Nginx or Apache to handle traffic on ports 80 and 443 and forward it to Odoo on port 8069.

1.  In the Cloud Console, go to **VPC network** > **Firewall**.
2.  Click **Create Firewall Rule**.
3.  **Name:** `allow-odoo-8069`
4.  **Targets:** `All instances in the network` (or specify your VM instance with tags)
5.  **Source IP ranges:** `0.0.0.0/0` (This allows traffic from any IP. For more security, restrict it to your IP address.)
6.  **Protocols and ports:** Check **TCP** and enter `8069`.
7.  Click **Create**.

You should now be able to access your Odoo instance by navigating to `http://<your_vm_external_ip>:8069` in your browser.
