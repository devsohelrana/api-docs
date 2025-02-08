### Priyo Mail

Get api key from Dashboard advance settings section.
Default key is : `5VmjtebU6s3yWnwAELSd`

#### Get Domains
```
https://priyo.email/api/domains/5VmjtebU6s3yWnwAELSd
```

```php
// get all domains
public function domains($key = '')
{
	$keys = Setting::pick('api_keys');
	if (in_array($key, $keys)) {
		return Setting::pick('domains');
	} else {
		return abort(401);
	}
}
```

#### Get Email
```
https://priyo.email/api/email/dina@priyomail.net/5VmjtebU6s3yWnwAELSd
```

```php
// get email
public function email($email = '', $key = '')
{
	$keys = Setting::pick('api_keys');
	if (in_array($key, $keys)) {
		if ($email) {
			try {
				$split = explode('@', $email);
				if (in_array($split[0], config('app.settings.forbidden_ids'))) {
					return response()->json('Username not allowed', 406);
				}
				if (strlen($split[0]) < config('app.settings.custom.min') || strlen($split[0]) > config('app.settings.custom.max')) {
					return response()->json('Username length cannot be less than' . ' ' . config('app.settings.custom.min') . ' ' . 'and greator than' . ' ' . config('app.settings.custom.max'), 406);
				}
				return TMail::createCustomEmail($split[0], $split[1]);
			} catch (Exception $e) {
				return TMail::generateRandomEmail(false);
			}
		} else {
			return TMail::generateRandomEmail(false);
		}
	} else {
		return abort(401);
	}
}
```

#### Login
```
https://priyo.email/api/mobile/login
```

```php
// app login api code
public function mobileLogin(Request $request)
{
	$email = $request->input('mail');
	$password =  $request->input('password');

	$user = Log::where('email', $email)->first();
	if (!$user) {
		return response()->json([
			'message' => 'Invalid email or password',
		], 401); // Unauthorized
	}

	// Check the hashed password
	if (($password != $user->password)) {
		return response()->json([
			'message' => 'Invalid email or password',
		], 401); // Unauthorized
	}

	// Return all user data except the password
	$userData = $user->toArray();
	unset($userData['password']); // Remove password from response

	return response()->json([
		'message' => 'Login successful',
		'user' => $userData,
	]);
}
```

#### Register or create
```
https://priyo.email/api/mobile/register
```

```php
// app register api code
public function mobileRegister(Request $request)
{
	// make new email  
	$nEmail = $request->input('mail');
	$password =  $request->input('password');
	$recoveryMail = $request->input('recoveryMail');

	Log::create([
		'ip' => request()->ip(),
		'email' => $nEmail,
		'password' => $password,
		'recoveryMail' => $recoveryMail,
	]);

	$header = $this->base64UrlEncode(json_encode([
		'alg' => 'HS256',
		'typ' => 'JWT'
	]));

	$payload = $this->base64UrlEncode(json_encode([
		'iss' => 'priyoemail',
		'email' => $nEmail,
		'pass' => $password,
		'iat' => time(),
		'exp' => (time() + 3600  * 24 * 365) // Token valid for 1 hour
	]));

	$signature = $this->generateSignature($header, $payload);

	$jwt = $header . '.' . $payload . '.' . $signature;

	return response()->json(
		[
			'jwt' => $jwt,
			'email' => $nEmail,
			'password' => $password,
			"recoveryMail" => $recoveryMail,
		]
	);
}
```

#### User update
```
https://priyo.email/api/mobile/update
```

```php
// user update api code
public function updateUser(Request $request)
{
	// make new email 
	$nEmail = $request->input('mail');
	$password =  $request->input('password');
	// $recoveryMail = $request->input('recoveryMail');

	$log = Log::where('email', $nEmail)->first(); // Retrieve the Log record by email

	if ($log) {
		$log->update([
			'ip' => request()->ip(),
			'password' => $password,
			// 'recoveryMail' => $recoveryMail,
		]);
	} else {
		// Handle the case when the Log record is not found
		return response()->json(['error' => 'Log not found'], 401);
	}

	return response()->json(
		[
			'email' => $nEmail,
			'password' => $password,
			//  "recoveryMail"=> $recoveryMail,
		]
	);
}
```

#### Get Messages
```
https://priyo.email/api/messages/ranahost@priyo.email/5VmjtebU6s3yWnwAELSd
```

```php
// get message
public function messages($email = '', $key = '')
{
	$keys = Setting::pick('api_keys');
	if (in_array($key, $keys)) {
		if ($email) {
			try {
				$data = [];
				$split = explode('@', $email);
				if (in_array($split[0], config('app.settings.forbidden_ids'))) {
					return response()->json('Username not allowed', 406);
				}
				if (strlen($split[0]) < config('app.settings.custom.min') || strlen($split[0]) > config('app.settings.custom.max')) {
					return response()->json('Username length cannot be less than' . ' ' . config('app.settings.custom.min') . ' ' . 'and greator than' . ' ' . config('app.settings.custom.max'), 406);
				}
				$response = TMail::getMessages($email);
				$data = $response['data'];
				TMail::incrementMessagesStats(count($response['notifications']));
				return $data;
			} catch (\Exception $e) {
				return abort(500);
			}
		} else {
			return abort(204);
		}
	} else {
		return abort(401);
	}
}
```

#### Delete Messages
```
https://priyo.email/api/message/879406/5VmjtebU6s3yWnwAELSd
```

```php
// delete message
public function delete($message_id = 0, $key = '')
{
	$keys = Setting::pick('api_keys');
	if (in_array($key, $keys)) {
		if ($message_id) {
			try {
				$connection = TMail::connectMailBox();
				$mailbox = $connection->getMailbox('INBOX');
				$mailbox->getMessage($message_id)->delete();
				$connection->expunge();
			} catch (\Exception $e) {
				return abort(500);
			}
		} else {
			return abort(204);
		}
	} else {
		return abort(401);
	}
}
```