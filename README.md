# How To Deploy Laravel on Zerops with Apache and PostgreSQL

## Introduction

Laravel is one of the most popular PHP frameworks, known for its elegant syntax and robust features. In this tutorial, you'll learn how to deploy a Laravel application on Zerops, a modern cloud platform designed for developers. We'll set up a complete environment with Apache web server and PostgreSQL database, though Zerops also supports Nginx and MySQL configurations if those better suit your needs.

For database management, we'll use PostgreSQL service provided by Zerops. Using built-in VPN functionality, you'll be able to connect to the database directly from your local environment without needing PostgreSQL installed locally. This means you can use your favorite database tools and run Laravel migrations while working with the database in Zerops.

By the end of this guide, you'll have:
- A fresh Laravel installation running locally
- A configured Zerops project with Apache and PostgreSQL
- Zero downtime deployment setup with environment variables
- A production-ready Laravel application accessible via Zerops subdomain
- Secure VPN access to your PostgreSQL database

> 💡 **Quick Start:** If you want to skip the tutorial and deploy a similar setup right away, you can find a complete example in our [Laravel Recipe Repository](https://github.com/zeropsio/recipe-laravel-minimal) and deploy it directly using the "Deploy on Zerops" button.

## Prerequisites

Before you begin, ensure you have:

- A Zerops [account](https://app.zerops.io)
- Git installed (required for deployment)

> 📝 **Note:** While we'll use PHP 8.4 on Zerops, your local PHP version can be any version supported by your Laravel project.

## Step 1 — Creating a New Laravel Project

Let's start by creating a fresh Laravel project:

```bash
composer create-project laravel/laravel zerops-laravel
cd zerops-laravel
```

> 💡 **Note:** Instead of setting up a local database, we'll later configure your local Laravel to connect directly to the PostgreSQL database in Zerops through a secure VPN connection. This approach simplifies our development workflow and ensures we're always working with the actual database.

Now, let's verify everything works locally. Start Laravel's development server:

```bash
php artisan serve
```

Visit `http://localhost:8000` in your browser. You should see Laravel's welcome page.

> 💡 **Note:** While we're using Laravel's built-in server for simplicity, you can use any local development setup you prefer (Valet, Sail, XAMPP, etc.).


If you see this page, great! Your local setup is working correctly. Press `Ctrl+C` in your terminal to stop the development server.

## Step 2 — Setting Up Your Zerops Project

First, we'll need to install and configure the Zerops Command Line Interface (zcli).

### Installing zcli

Choose your operating system:

**Linux/MacOS:**
```bash
curl -L https://zerops.io/zcli/install.sh | sh
```

**Windows** (run in PowerShell):
```powershell
irm https://zerops.io/zcli/install.ps1 | iex
```

> 💡 **Note:** If you prefer alternative installation methods (like npm), check the [zcli documentation](https://github.com/zeropsio/zcli).

### Logging into zcli

1. Get your access token from [Zerops Token Management](https://app.zerops.io/settings/token-management)

2. Login using your token (one-time setup):
```bash
zcli login PLACE_TOKEN_HERE
```

### Creating Project Configuration

> 🔍 **Note:** Zerops automatically creates network connections between services in the same project. Your `app` service will be able to reach the `db` service using the hostname `db`.

Create a new file called `zerops-project-import.yml` with the following content:

```yaml
#yamlPreprocessor=on
project:
  name: project
services:
  - hostname: app
    type: php-apache@8.4
    envSecrets:
      # yamlPreprocessor feat: generates a random 32 char and stores it
      APP_KEY: <@generateRandomString(<32>)>
    
  - hostname: db
    type: postgresql@16
    mode: NON_HA
```

> 💡 **Note:** The `#yamlPreprocessor=on` directive enables Zerops' YAML preprocessing for this import file, allowing us to use dynamic values and built-in functions like `generateRandomString`.

### Creating Project using zcli

Now create the project by running:
```bash
zcli project project-import zerops-project-import.yml
```

### Alternative: Creating Project via GUI

You can also create and configure your project through the Zerops dashboard:

1. Log into your [Zerops Dashboard](https://app.zerops.io)
2. Click "Add new project"
3. Enter a project name (e.g., "laravel-zerops-project") and click "Create project"

Then add the required services:

1. **PHP + Apache Service:**
   - Click "Add Service" and select "PHP+Apache"
   - Set hostname to "app"
   - Keep all other settings as default

2. **PostgreSQL Service:**
   - Click "Add Service" and select "PostgreSQL"
   - Set hostname to "db"
   - Keep all other settings as default

> 💡 **Note:** All services within your project are automatically connected via a private network and can access each other using their hostnames.

## Step 3 — Configuring Your Application

Create a `zerops.yml` configuration file in your project root:

```yaml
zerops:
  - setup: app
    build:
      base:
        - php@8.4
      buildCommands:
        - composer install --ignore-platform-reqs
      deployFiles: ./
      cache:
        - vendor
        - composer.lock
    deploy:
      readinessCheck:
        httpGet:
          port: 80
          path: /up
    run:
      base: php-apache@8.4
      envVariables:
        APP_NAME: "Laravel Zerops Demo"
        APP_DEBUG: false
        APP_ENV: production
        APP_URL: ${zeropsSubdomain}
        DB_CONNECTION: pgsql
        DB_HOST: db
        DB_PORT: 5432
        DB_DATABASE: db
        DB_USERNAME: ${db_user}
        DB_PASSWORD: ${db_password}
        LOG_CHANNEL: stack
        LOG_LEVEL: debug
      initCommands:
        - php artisan config:cache
        - php artisan route:cache
        - sudo -E -u zerops -- zsc execOnce ${appVersionId} -- php artisan migrate --force
        - php artisan optimize
      healthCheck:
        httpGet:
          port: 80
          path: /up
```

Let's break down some important parts of this configuration:

### Environment Variables

Zerops provides several built-in variables that you can use in your configuration:

```yaml
APP_URL: ${zeropsSubdomain}  # Automatically set to your app's Zerops URL
DB_USERNAME: ${db_user}      # Database credentials auto-injected
DB_PASSWORD: ${db_password}  # Securely managed by Zerops
```

These values are automatically injected by Zerops:
- `${zeropsSubdomain}` - Your application's Zerops URL
- `${db_user}` and `${db_password}` - Secure database credentials

### Safe Database Migrations
```yaml
sudo -E -u zerops -- zsc execOnce ${appVersionId} -- php artisan migrate --force
```
This special command ensures your database migrations run exactly once, even when running multiple containers. It's crucial for preventing duplicate migrations during scaling.

> ⚡ **Power User Tip:** You can also manage these configurations using Zerops' GUI or import them via zcli. The YAML approach gives you version control and reproducibility.

## Step 4 — Deploying Your Application

Now comes the exciting part - deploying your application to Zerops!

### Deploying Your Code

Initialize git in your project directory:
```bash
git init
```

> 💡 **Note:** Git is used to determine which files should be sent to Zerops. You don't need to commit any changes, but git initialization is required for the deployment process.

Push your code to Zerops:
```bash
zcli push
```

You'll see output similar to:
```
Preparing deployment package...
Uploading files...
Build triggered successfully!
```

### Monitoring the Deployment

1. Go to the app service page
2. In the top-left corner, you'll see a circle with "running process" text
3. Click it to see the build progress overview
4. Click "Open pipeline detail" button to view the detailed build process

You'll see the deployment progress with timing for each step:
- Initializing build container
- Running build commands from zerops.yml
- Creating app version and upgrading PHP+Apache service

The entire process usually takes less than a minute to complete.

## Step 5 — Verifying Your Deployment

Once deployment completes, let's verify everything works:

1. Go to your project's app service
2. Click on "Public access & internal ports"
3. Find the "Public Access through zerops.app Subdomain" section
4. Toggle "Enable Zerops Subdomain Access"
5. Click the generated URL (e.g., https://app-xxx.prg1.zerops.app) to view your application

> 🔍 **Note:** The Zerops subdomain is perfect for testing and development, but for production, you should set up your own domain under "Public Access through Your Domains".

### Testing Database Connectivity

Let's create a quick route to test database connectivity. Add this to your `routes/web.php`:

```php
Route::get('/db-test', function () {
    try {
        DB::connection()->getPdo();
        return 'Database connected successfully!';
    } catch (\Exception $e) {
        return 'Database connection failed: ' . $e->getMessage();
    }
});
```

Deploy this change:
```bash
zcli push
```

Visit your-app-url/db-test to verify database connectivity.

## Accessing Your Database Locally

Once your application is deployed, you might want to access the database directly from your local machine. Zerops makes this easy with VPN access.

### Setting up VPN Access

1. Start the VPN connection:
```bash
zcli vpn up
```

2. Select your project when prompted

That's it! You now have direct access to all services in your project.

### Connecting to Database

To get your database credentials:
1. Go to the PostgreSQL service in your project
2. Click "Access details" button
3. Here you'll find all connection details including hostname, port, user, and password

Update your local `.env` file with these credentials:

```ini
DB_CONNECTION=pgsql
DB_HOST=db.zerops
DB_PORT=5432
DB_DATABASE=db
DB_USERNAME=db
DB_PASSWORD=[password from Access details]
```

> 🔒 **Security Note:** Never share or commit your database credentials. Always get them from the Zerops dashboard Access details section.

Now you can use your favorite database management tool or run artisan commands while working with the database in Zerops - no local PostgreSQL installation needed!

## Conclusion

Congratulations! 🎉 You've successfully deployed a Laravel application on Zerops with Apache and PostgreSQL. Your application is now running in a production-ready environment with automated builds and deployments.

### Additional Resources

- [Official Zerops Documentation](https://docs.zerops.io)
- [Laravel Recipe Repository](https://github.com/zeropsio/recipe-laravel-minimal)
- [zcli Documentation](https://github.com/zeropsio/zcli)

### Need Help?

- Join [Zerops Discord](https://discord.com/invite/WDvCZ54)
- Check out the [Zerops Blog](https://zerops.io/blog)
- Follow [@zeropsio](https://twitter.com/zeropsio) on Twitter
