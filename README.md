# zabbix-login-logger

Tested on Zabbix 6.0.19.

Adds logs to /var/log/apache2/error.log about web frontend logins

```
[Thu Jul 20 07:19:35.112119 2023] [php7:notice] [pid 2440044] [client 45.130.22.219:45356] Login failure: Admin, referer: https://xxx.yyy.zzz.www:443/
[Thu Jul 20 08:32:29.835759 2023] [php7:notice] [pid 2440043] [client 45.130.22.219:54744] Login failure: zabbix, referer: https://xxx.yyy.zzz.www:443/
```

Change CWebUser.php file located at /usr/share/zabbix/include/classes/user/CWebUser.php

Change function login

	public static function login(string $login, string $password): bool {
		try {
			self::$data = API::User()->login([
				'username' => $login,
				'password' => $password,
				'userData' => true
			]);

			if (!self::$data) {
				throw new Exception();
			}

			API::getWrapper()->auth = self::$data['sessionid'];

			if (self::$data['gui_access'] == GROUP_GUI_ACCESS_DISABLED) {
				error(_('GUI access disabled.'));
				throw new Exception();
			}

			if (isset(self::$data['attempt_failed']) && self::$data['attempt_failed']) {
				CProfile::init();
				CProfile::update('web.login.attempt.failed', self::$data['attempt_failed'], PROFILE_TYPE_INT);
				CProfile::update('web.login.attempt.ip', self::$data['attempt_ip'], PROFILE_TYPE_STR);
				CProfile::update('web.login.attempt.clock', self::$data['attempt_clock'], PROFILE_TYPE_INT);
				if (!CProfile::flush()) {
					return false;
				}
			}
			// Registro de inicio de sesión exitoso
			self::writeLoginLog($login, true);
			
			return true;
		}
		catch (Exception $e) {
			// Registro de inicio de sesión fallido
			self::writeLoginLog($login, false);
			
			self::setDefault();
			return false;
		}
	}

next, add the funcion

	// Funcion logger
	private static function writeLoginLog(string $login, bool $success): void {
		$logMessage = $success ? 'Login success: ' : 'Login failure: ';
		$logMessage .= $login;

		error_log($logMessage);
	}
