use Rex -feature => ['1.0'];

user "alpha6";
private_key "~/.ssh/id_rsa";
public_key "~/.ssh/id_rsa.pub";
key_auth;

group myservers => "alpha6.ru";

desc "Deploy the blog on alpha6.ru";
task "deploy", group => "myservers", sub {
   run "cd /srv/www/alpha6.ru && git pull origin gh-pages";
};
