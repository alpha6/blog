---
tags: perl, Net::SMTP, email
title: Отправляем емейл из Perl с авторизацией и html
---


	  my $message = "<html><body>Тестовое письмо</body></html>";
	  # SMTP credentials
    my $smtp_host = 'smtp.server.address';
    my $smtp_user = 'user_name';
    my $smtp_pass = 'password';


    my $debug = 1;

    # mail properties
    my $mail_from = 'user@example.com';
    my $mail_to = 'recipient@example.com';
    my $mail_subject = 'Тестовое письмо';
    my $mail_headers = "From: $mail_from\n".
    "To: $mail_to\n".
    "Subject: ".encode('MIME-Header',$mail_subject)."\n".
    "MIME-Version: 1.0\n".
    "Content-type: text/html; charset=UTF-8\n".
    "Content-Transfer-Encoding: base64\n\n";
    my $mail_body = $message;

    # send the email
    my $smtp = Net::SMTP->new($smtp_host, Debug => $debug) or die "cannot connect to server";
    $smtp->auth($smtp_user,$smtp_pass) or die "could not authenticate";
    $smtp->mail($mail_from);
    $smtp->to($mail_to);
    $smtp->data();
    $smtp->datasend($mail_headers);
    $smtp->datasend(encode_base64(encode('UTF-8', $mail_body)));
    $smtp->dataend();
    $smtp->quit;
