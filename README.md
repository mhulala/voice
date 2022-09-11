# Voice
Call routing with voicemail (using **SignalWire** API)

---

When I got my new mobile phone number, I didn’t expect to get too many “wrong number” calls. However, the reality was much different. While most of these calls ended within several weeks, there is one old lady who calls several times every day. And she is still looking to talk to some other Michael, not willing to stop calling my number. While blocking her (mobile and landline) numbers is very easy on every phone, the blocking doesn’t work when the phone is off. So my voicemail gets full very quickly, and most importantly, it was very annoying to remove her voicemail daily. The phone company was unwilling to block her numbers in their system without a court order, so I decided to take care of that on my end.

GSM MMI codes allow diverting phone calls to different numbers. In my case, I used the code `**004*number#` for conditional call forwarding. SignalWire platform offers cheap numbers (20¢ per number a month) and an excellent Twilio-like API. While their *“called”* parameter is not working as supposed to, thankfully, I was able to use *“SipHeader_Diversion”* information to apply a fix to get a proper *“called”* value.

I coded my application with flexibility in mind. The script uses the **MySQL** database, allowing routing parameters to meet comprehensive action options. Based on who calls what number, the call can be rejected, forwarded to a different number, answered with a busy signal, answered by an audio or text-to-speech message (or both), with an option to leave and record a message. The text-to-speech engine supports men’s and women’s voices in many languages, and audio messages can be supplied in many standard forms, including the most common MP3 and WAV formats.

The application written in **Perl** was tested on Linux machines with the support of **.htaccess**. This configuration file has to include the following lines:

    <FilesMatch '^[^.]+$'>
      SetHandler cgi-script
    </FilesMatch>

    <IfModule rewrite_module>
      RewriteEngine On
      RewriteBase /
      RewriteRule ^voice\.(mp3|wav|aif|gsm|ulaw)$ /voice [QSA,END]
    </IfModule>

Indeed, the configuration can be modified to your needs. My script does not use a .pl extension, so in .htaccess, it was necessary to set files without extension as CGI scripts. The second part of the .htaccess configuration redirects related audio files to the script.

I use **Microsoft Graph** API to send an email message with recorded voicemail to set the recipient. Similarly to database credentials, Microsoft Graph credentials have to be entered into the code: 

    use constant mysql_database          => '##########';
    use constant mysql_username          => '##########';
    use constant mysql_password          => '##########';
    use constant ms_graph_email_address  => '##########';
    use constant ms_graph_client_id      => '##########';
    use constant ms_graph_client_secret  => '##########';
    use constant ms_graph_get_token_url  => 'https://login.microsoftonline.com/##########/oauth2/v2.0/token';
    use constant ms_graph_send_mail_url  => 'https://graph.microsoft.com/v1.0/users/##########/sendmail';

To store logs files, correct paths have to be set as well:

    use constant log_file_path           => '/var/www/vhosts/##########';
    use constant err_file_path           => '/var/www/vhosts/##########';

Last but not least, the MySQL database and related tables have to be created as follows:

    CREATE TABLE `accounts` ( `aid` char(36) NOT NULL, `status` enum('disabled','enabled') NOT NULL DEFAULT 'enabled', PRIMARY KEY (`aid`), UNIQUE KEY `aid_UNIQUE` (`aid`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
    CREATE TABLE `numbers` ( `number` char(16) NOT NULL, `aid` char(36) NOT NULL, `status` enum('disabled','enabled') NOT NULL DEFAULT 'enabled', `service` enum('busy','fax','diversion','routing','voicemail') NOT NULL DEFAULT 'busy', `destination` char(16) DEFAULT NULL COMMENT 'Column `destination` is disregarded unless `service` is set to ''diversion''.', PRIMARY KEY (`number`), UNIQUE KEY `number_UNIQUE` (`number`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='All custom routings are disregarded for `mode` set to ''strict''. Column `forwardTo` is disregarded for services other than ''forward''.'
    CREATE TABLE `routings` ( `id` int DEFAULT NULL, `from` char(16) DEFAULT NULL, `to` char(16) NOT NULL, `called` char(16) DEFAULT NULL COMMENT 'A NULL value in column `called` represents any (other than explicitly listed) values.', `status` enum('disabled','enabled') NOT NULL DEFAULT 'enabled', `service` enum('busy','diversion','voicemail') NOT NULL DEFAULT 'busy', `destination` char(16) DEFAULT NULL COMMENT 'Column `destination` is disregarded unless `service` is set to ''diversion''.', UNIQUE KEY `UNIQUE` (`to`,`from`,`called`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='A NULL value in column `called` represents any (other than explicitly listed) values. Column `forwardTo` is disregarded for services other than ''forward''.'
    CREATE TABLE `voicemails` ( `id` int DEFAULT NULL, `notes` varchar(256) DEFAULT NULL, `number` char(16) NOT NULL, `from` char(16) DEFAULT NULL, `called` char(16) DEFAULT NULL, `status` enum('disabled','enabled') NOT NULL DEFAULT 'enabled', `greetingPause` tinyint unsigned NOT NULL DEFAULT '0', `greetingOrder` enum('audio-text','text-audio') NOT NULL DEFAULT 'audio-text', `greetingAudioMedia` longblob, `greetingAudioMediaMimeType` enum('audio/mpeg','audio/wav','audio/wave','audio/x-wav','audio/aiff','audio/x-aifc','audio/x-aiff','audio/x-gsm','audio/gsm','audio/ulaw') NOT NULL DEFAULT 'audio/mpeg', `greetingText` varchar(1024) DEFAULT 'The party you are trying to reach is not available to take your call. Please leave a message after the tone.', `greetingTextVoice` enum('man','woman') NOT NULL DEFAULT 'woman', `greetingTextLanguage` enum('arb','ar-AR','ar-XA','bn-IN','yue-HK','cmn-CN','cmn-TW','zh-CN','cs-CZ','cy-GB','da-DK','de','de-DE','en','en-AU','en-CA','en-GB','en-GB-WLS','en-IN','en-US','es','es-ES','es-MX','es-US','fil-PH','fi-FI','fr','fr-CA','fr-FR','el-GR','gu-IN','hi-IN','hu-HU','is-IS','id-ID','it','it-IT','ja-JP','kn-IN','ko-KR','ml-IN','morse','nb-NO','nl-NL','pl-PL','pt-BR','pt-PT','ro-RO','ru-RU','sk-SK','sv-SE','ta-IN','te-IN','th-TH','tr-TR','uk-UA','vi-VN') NOT NULL DEFAULT 'en', `recording` enum('yes','no') NOT NULL DEFAULT 'yes', `recordingBeep` enum('true','false') NOT NULL DEFAULT 'true', `recordingMaxLength` smallint unsigned NOT NULL DEFAULT '120', `recordingTimeout` tinyint unsigned NOT NULL DEFAULT '10', `recordingTrim` enum('do-not-trim','trim-silence') NOT NULL DEFAULT 'do-not-trim', `recipientName` varchar(256) DEFAULT NULL, `recipientEmailAddress` varchar(256) DEFAULT NULL, `locale` varchar(32) NOT NULL DEFAULT 'en_US', `timezone` varchar(32) NOT NULL DEFAULT 'UTC', `datetimeFormat` varchar(64) NOT NULL DEFAULT '%A %b-%d-%Y, %H:%M:%S', `regexList` blob, `mp3_emailType` enum('Text','HTML') NOT NULL DEFAULT 'HTML', `mp3_emailSubject` varchar(256) NOT NULL DEFAULT 'You''ve got a new voicemail%[[FROM| from $$]]%', `mp3_emailBody` blob, `url_emailType` enum('Text','HTML') NOT NULL DEFAULT 'HTML', `url_emailSubject` varchar(256) NOT NULL DEFAULT 'You''ve got a new voicemail%[[FROM| from $$]]%', `url_emailBody` blob, `err_emailType` enum('Text','HTML') NOT NULL DEFAULT 'HTML', `err_emailSubject` varchar(256) NOT NULL DEFAULT 'Error receiving voicemail', `err_emailBody` blob, UNIQUE KEY `UNIQUE` (`id`,`number`,`from`,`called`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
    CREATE TABLE `logs` ( `cid` char(36) NOT NULL, `aid` char(36) NOT NULL, `mid` char(36) DEFAULT NULL, `timestamp` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, `from` char(16) NOT NULL, `to` char(16) NOT NULL, `called` char(16) DEFAULT NULL, `response` enum('busy','diversion','fax','rejection','voicemail') NOT NULL, `destination` char(16) DEFAULT NULL COMMENT 'Column `destination` is valid only if `response` is set to ''diversion''.', PRIMARY KEY (`cid`), UNIQUE KEY `cid_UNIQUE` (`cid`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
    CREATE TABLE `recordings` ( `mid` char(36) NOT NULL, `media` longblob, `mediaName` varchar(256) DEFAULT NULL, `mediaMimeType` set('audio/mpeg') NOT NULL DEFAULT 'audio/mpeg', PRIMARY KEY (`mid`), UNIQUE KEY `mid_UNIQUE` (`mid`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

Since Microsoft Graph API requires the Base64 encoding for attachments, the stored audio files in the MySQL database are also held with the same encoding.
