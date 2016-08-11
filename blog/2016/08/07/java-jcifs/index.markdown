---
tags: java, jcifs, samba
title: Работаем с самбой из Java с помощью JCIFS
---
# Простые примеры работы с самбой при помощи JCIFS

  import jcifs.Config;
  import jcifs.smb.*;

  import java.io.*;

  public class SambaTest {
    // Нормальный конструктор я тут делать не буду, для тестового примера это не нужно
    static final String USER_NAME = "username";
    static final String PASSWORD = "password";
    static final String DOMAIN = "user_domain";

    // Путь к сетевой папке с которой будем работать
    static final String NETWORK_FOLDER = "smb://server/share/";


    // Копируем файл на шару.
    // К сожалению SmbFile ничего не знает о методе copyTo из File, так что придется этот метод эмулировать руками. Халява не прокатила :(
    // В обратную сторону все тоже самое, только потоки будут в обратную сторону.
    public boolean copyFileToSamba(String srcFilePath, String destPath) {
        boolean successful = false;
        try{
            // Создаем объект аутентификатор
            NtlmPasswordAuthentication auth = new NtlmPasswordAuthentication(DOMAIN, USER_NAME, PASSWORD);

            // Читаем содержимое исходного файла
            File srcFile = new File(srcFilePath);
            InputStream localFile = new FileInputStream(srcFile);

            // Создаем объект для потока куда мы будем писать наша файл
            SmbFileOutputStream destFileName = new SmbFileOutputStream(new SmbFile(destPath+File.separator+srcFile.getName(), auth));

            // Ну и копируем все из исходного потока в поток назначения.
            BufferedReader brl = new BufferedReader(new InputStreamReader(localFile));
            String b = null;
            while((b=brl.readLine())!=null){
                destFileName.write(b.getBytes());
            }
            destFileName.flush();


            successful = true;
        } catch (Exception e) {
            successful = false;
            e.printStackTrace();
        }
        return successful;
    }

    // Читаем
    public boolean readShareContent() {
        boolean successful = false;
        try{
            // Создаем объект для аутентификации на шаре
            NtlmPasswordAuthentication auth = new NtlmPasswordAuthentication(DOMAIN, USER_NAME, PASSWORD);
            String path = NETWORK_FOLDER;

            // Ресолвим путь назначения в SmbFile
            SmbFile baseDir = new SmbFile(path, auth);

            // Вычитываем все содержимое шары в массив
            SmbFile[] files = baseDir.listFiles();

            // Делаем что-нибудь со списком файлов
            for (int i = 0; i < files.length; i++) {
              SmbFile file = files[i];
              if (file.isDirectory()) {
                    System.out.println("Is DIR: "+file.toString());
                    continue;
              } else {
                    System.out.println("Is FILE: "+file.toString());
              }
            }

            successful = true;
        } catch (Exception e) {
            successful = false;
            e.printStackTrace();
        }
        return successful;
      }
    }

Если все работы с шарой невероятно тупят - то нужно сменить режим ресолвинга сервера и шары.

В конструктор добавляем следующее:

  Config.setProperty( "jcifs.resolveOrder", "DNS");

Этим мы указываем что ресолвить имя сервера мы будем ТОЛЬКО через DNS. Можно туда добавить всякие NetBIOS и прочее, но на практике я с необходимостью это делать не сталкивался. Естественно, что если у вас в сети нет локального DNS сервера который будет ресолвить именя локальным машин - то механизм надо сменить на другой. Выбор механизмов ресолвинга будет проходить в порядке указанном в конфиге.

Если предполагается копирование больших бинарных файлов, то код копирования надо заменить с:

  BufferedReader brl = new BufferedReader(new InputStreamReader(localFile));
  String b = null;
  while((b=brl.readLine())!=null){
    destFileName.write(b.getBytes());
  }

на:

  byte[] buffer = new byte[1024000];
  int noOfBytes = 0;

  while ((noOfBytes = localFile.read(buffer)) != -1) {
    destFileName.write(buffer, 0, noOfBytes);
  }

Если этого не сделать - то большие файлы будут копироваться криво.
