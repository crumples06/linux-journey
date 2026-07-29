# Week-7
*27/07/26 - 02/08/26*

## curl command
It is used to transfer data between a system and a server using different network protocols.

- `curl [URL]` like say `curl https://geeksforgeeks.com/` will print the contents of that webpage on stdout. It's not very useful as it is as the content is not formatted.

- - The `-I` flag only fetches headers from the webpage. It is much easier to understand.

- I can write multiple URL's by `curl https://site.{one, two, three}.com`, and numeric sequences using `curl ftp://ftp.example.com/file[1-20].jpeg`.

- `-#` option will display a progress bar, and `--silent` disables it.

- To download files from authenticated FTP server, `curl -u {username}:{password} [FTP_URL]`. If i add a `-T {filename}` flag after password, i can upload files.

- One thing i found pretty cool was that i can send emails using this command. `curl --url [SMPT URL] --mail-from [sender mail] --mail-rcpt [receiver mail] -n --ssl-readq -u {email}:{password} -T [mail_text_file]`. The `[SMTP URL]` is the address of the mail server that will relay the message, like for example for gmail it will be `smtp://smtp.gmail.com:587`.

- curl can also be used to test REST APIs, i can perform GET,POST,PUT operations using this command. example: `curl -X GET https://api.sampleapis.com/coffee/hot` and `curl -X POST -d "key1=value1&key2=value2" https://api.sampleapis.com/coffee/hot`. The `id` flag is used to send data with the request.


