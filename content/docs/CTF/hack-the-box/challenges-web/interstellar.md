+++
title = "Interstellar"
date = 2024-01-14T07:07:07+01:00
draft = false
+++

# Interstellar

{{< figure src="/assets/interstellar/banner.png" >}}

Interstellar is a medium-level web challenge from Hack The Box.

## Setup test environment

For the challenge, the source code is provided along with Docker configuration files, allowing us to easily set up the environment for the application.

Simply spinning up Docker is enough to have a functional environment running.

{{< figure src="/assets/interstellar/app-running.png" >}}

## Taint Analisys
In the current phase, the objective is to use the application to identify its functionalities and endpoints, as well as to leverage the source code to examine the implementation of these endpoints and search for potential attack vectors, thereby identifying the application's attack surface. Caido will be used as a proxy to record the endpoints identified during interaction with the system and to create a sitemap.

{{< figure src="/assets/interstellar/proxy-sitemap.png" >}}

Through interaction with the system, the following functionalities were identified: account registration, login, edit name and communication device.

The name-change functionality only works if the request originates from localhost.

{{< figure src="/assets/interstellar/change-name-interface.png" >}}

{{< figure src="/assets/interstellar/change-name-proxy.png" >}}

```php
elseif($action =="edit"){
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            if ($_SERVER['REMOTE_ADDR'] != '127.0.0.1') {
                $smarty->assign('error', "Only localhost can use this function!");
                $smarty->display('index.tpl');
                exit();
            }
            $new_name = $_REQUEST['new_name'] ?? '';
            try {
                $query = "CALL editName(?, ?)";
                $stmt = $conn->prepare($query);
                $stmt->bind_param("is", $id, $new_name);
                $stmt->execute();
    
                if ($stmt->affected_rows > 0) {
                    $_SESSION['name'] = $new_name;
                    $smarty->assign('message', "Done!");
                    $smarty->display('index.tpl');
                    exit();
                } else {
                    $smarty->assign('error', "Failed!");
                    $smarty->display('index.tpl');
                    exit();
                }
                $stmt->close();
            } 
```

The communication functionality uses a URL as a parameter, but it only accepts URLs for motherland.com.

{{< figure src="/assets/interstellar/communicate-interface.png" >}}

{{< figure src="/assets/interstellar/communicate-proxy.png" >}}

```php
  if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        $url = $_POST['url'];
        $data = $_POST['data'] ?? [];
        if(filter_var($url, FILTER_VALIDATE_URL)) {
            $parsedUrl= parse_url($url);
            if(preg_match('/motherland\.com$/', $parsedUrl['host'])) {    //currently just support call to your mother land 
                try {
                    $ch = curl_init();
                    $sessCookie = isset($_COOKIE['PHPSESSID']) ? $_COOKIE['PHPSESSID'] : '';
                    curl_setopt($ch, CURLOPT_URL, $parsedUrl['host']);
                    curl_setopt($ch, CURLOPT_POST, true);
                    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
                    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
                    curl_setopt($ch, CURLOPT_HTTPHEADER, [
                        "Cookie: PHPSESSID=$sessCookie;"
                    ]);
                    curl_setopt($ch, CURLOPT_TIMEOUT, 1);
                    $response = curl_exec($ch);
        
                    if (curl_errno($ch)) {
                        $error = 'cURL Error: ' . curl_error($ch);
                    } else {
                        $error = null;
                    }
        
                    curl_close($ch);
        
        
                    $smarty->assign('response', $response ?? '');
                    $smarty->assign('error', $error ?? '');
        
                    $smarty->display('communicate.tpl');
                } catch (Exception $e) {
                    $smarty->assign('error', 'An error occurred: ' . $e->getMessage());
                    $smarty->display('communicate.tpl');
                }
            }else{
                $smarty->assign('error','Wrong URL!');
                $smarty->display('communicate.tpl');
            }
        }else{
            $smarty->assign('error','Failed when parsing URL!');
            $smarty->display('communicate.tpl');
        }
```

## SSRF

By examining the `communicate` function, it can be seen that validation is performed using `parse_url` and regex. If the host extracted via `parse_url` passes validation, it is used as a parameter for cURL, acting as a sink for SSRF.  It is worth noting that, due to the cURL configuration, this SSRF triggers a POST request containing attacker-controlled parameters in the body, along with the session cookie.

The only barrier to this SSRF is the aforementioned validation, so the current objective is to find ways to bypass this validation.

This article demonstrates some ways to bypass `parse_url` using regex, which is useful for the current scenario: https://medium.com/@themiddleblue/php-ssrf-techniques-9d422cb28d51.

By using one of the techniques from the article along with a webhook to receive the request, it was possible to confirm the SSRF.

{{< figure src="/assets/interstellar/ssrf-poc-proxy.png" >}}

{{< figure src="/assets/interstellar/ssrf-canary.png" >}}


This makes it possible to access the edit-name function, which was previously only accessible via localhost.

{{< figure src="/assets/interstellar/ssrf-edit-name.png" >}}

{{< figure src="/assets/interstellar/edited-name.png" >}}

## SQL Injection

The remaining features operate using bind_params and stored procedures.

Bind parameters typically neutralize SQL injection vectors; however, an analysis of the stored procedures reveals that the `searchUser` procedure follows a different pattern.

```sql
CREATE PROCEDURE searchUser(IN name VARCHAR(255))
BEGIN
    SET @sql = CONCAT('SELECT * FROM users WHERE name = \'', name, '\'');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //
```

The value received via a parameter in the procedure, which, despite being validated as a string by `bind_param`, is concatenated into the SQL query, grants us control over the query and renders the sink exploitable.

Now, it is necessary to identify where the `searchUser` procedure is called.

```php
$name = isset($_SESSION['name']) ? $_SESSION['name'] : null;
$id = isset($_SESSION['id']) ? $_SESSION['id'] : null;
$action = $_REQUEST['action'];

$smarty = getSmarty();
if ($id) {
    if(empty($action)){
        try {
            $query = "CALL searchUser(?)";
            $stmt = $conn->prepare($query);
            $stmt->bind_param("s", $name);
            $stmt->execute();
            if ($stmt->errno) {
                http_response_code(500);
                echo "SQL Error: " . $stmt->error;
                exit;
            }     
            $result = $stmt->get_result();
    
            $user = $result->fetch_object();
            if (!$user) {
                throw new Exception("No user found.");
            }
            $planet=$user->planet;
            $planet_emoji = pick_emoji($planet);
            $smarty->assign('planet', $planet_emoji);
            $smarty->assign('name', $name);
    
            $smarty->display('index.tpl');
    
        } catch (Exception $e) {
            
            $smarty->assign('message', 'An error occurred: ' . $e->getMessage());
            $smarty->assign('name', $name);
            $smarty->display('index.tpl');
        }
```

The main route, index.php, is what calls the `searchUser` procedure. It is important to note, however, that the value of the `name` parameter is derived from the session, rather than from a user-controlled parameter. Nevertheless, the previously discovered SSRF serves as the vector that allows us to change the `name` value to a payload designed to exploit the identified SQL injection sink.

The flag is located in the system root with a random filename, not in the database, so the objective is RCE (Remote Code Execution) rather than data exfiltration via SQL injection.

{{< figure src="/assets/interstellar/flag-system.png" >}}


The most direct way to exploit an SQL injection to achieve command execution is to write a webshell into the application's webroot.

{{< figure src="/assets/interstellar/injecting-web-shell.png" >}}

When accessing the index.php route, an error is visible on the screen instead of the standard response.

{{< figure src="/assets/interstellar/index-error.png" >}}

This indicates that the injected payload interfered with the executed query, meaning the web shell must have been written.
By accessing `/shell.php?cmd=id`, the command output appears on the screen, RCE successful.

{{< figure src="/assets/interstellar/webshell-id.png" >}}


## Exploit Chain

With the vulnerabilities identified, they can be combined to read the flag.
The attack plan is as follows:
1. Use SSRF to access the restricted name-change functionality.
2. Change the name to an SQL injection payload that will write a webshell to the application's webroot.
3. Use the webshell to read the flag.

Replacing the name with the SQLi payload:

{{< figure src="/assets/interstellar/prod-ssrf.png" >}}


Using a webshell to list the system root and identify the flag:

{{< figure src="/assets/interstellar/webshell-ls.png" >}}

Reading the flag:

{{< figure src="/assets/interstellar/webshell-cat.png" >}}
