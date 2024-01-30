Налаштування Jenkins CI/CD Pipeline на AWS EC2
==============================================

Вступ
-----

Цей README надає докладний посібник з налаштування Jenkins (CI/CD) pipeline на інстансі Amazon Web Services (AWS) EC2. Цей pipeline автоматизує процеси збірки, тестування та розгортання застосунку, і налаштований з нотифікаціями поштою та через Telegram. Процес налаштування включає конфігурацію Jenkins відповідно до [офіційного посібника Jenkins](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/) та налаштування pipeline за [цим посібником](https://medium.com/cloud-native-daily/setting-up-a-ci-cd-pipeline-process-with-jenkins-and-docker-in-aws-130a5e03192a), з використанням Jenkinsfile з цього другого посібника. Базовий jenkinsfile і тести можна знайти в цій репо в branch master.

Налаштування Jenkins на AWS EC2
-------------------------------

### Запуск EC2 Інстансу

1.  Створіть EC2 інстанс на AWS, обравши відповідний тип інстансу (наприклад, t3.small or medium).

### Встановлення Jenkins

1.  Підключіться до вашого EC2 інстансу за допомогою SSH чи ec2 connect:

    ```bash

    `ssh -i /path/to/key.pem ec2-user@your-ec2-public-dns.amazonaws.com`
    ```

2.  Встановіть Jenkins на EC2 інстанс. Детальні кроки наведені у [посібнику з встановлення Jenkins](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/). Це включає оновлення менеджера пакетів, встановлення Java та Jenkins.

### Конфігурація Jenkins

1.  Отримайте доступ до інтерфейсу Jenkins через `http://your-ec2-public-dns.amazonaws.com:8080`.

2.  Завершіть початкове налаштування Jenkins, включаючи розблокування Jenkins, встановлення рекомендованих плагінів та створення адміністративного користувача.

Налаштування Pipeline
---------------------

### Інтеграція Jenkins з Git
Тут я використовувала другий посібник, встановила докер на інстанс і всі потрібні credentials. Також під'єднала свою dockerhub репо.

1.  Створіть новий Pipeline job в Jenkins:

    -   Перейдіть до дашборду Jenkins і виберіть "New Item".
    -   Введіть назву для вашого pipeline, виберіть "Pipeline" і натисніть "OK".
2.  Додайте відкритий доступ до вашого Git репозиторію (для приватного треба паролі, але ми не платимо, тому все публічно):

    -   У налаштуваннях вашого Jenkins Pipeline job, додайте URL вашого Git репозиторію. Також в Definition - Pipiline script from SCM

### Налаштування Jenkinsfile

1.  Використайте Jenkinsfile з другого посібника:
    -   Jenkinsfile визначає стадії для збірки, тестування та розгортання.
    -   Для стадій збірки та тестування включені відповідні команди, що базуються на стеку проекту.
    -   В тому файлі ми пушимо image to dockerhub, але для розгортання застосунку в AWS ми можемл привнести такі зміни до jenkinsfile щоб використати AWS ECS:
      
   ```groovy
      stage('Push to ECR') {
            steps {
              withCredentials([usernamePassword(credentialsId: 'AWS_CREDENTIALS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                sh 'aws ecr get-login-password --region your-aws-region | docker login --username AWS --password-stdin your-ecr-repository-url'
                sh 'docker tag my-flask-app:latest your-ecr-repository-url/my-flask-app:latest'
                sh 'docker push your-ecr-repository-url/my-flask-app:latest'
              }
            }
          }
          stage('Deploy') {
            steps {
              withCredentials([usernamePassword(credentialsId: 'AWS_CREDENTIALS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                sh 'aws ecs update-service --cluster your-ecs-cluster-name --service your-ecs-service-name --force-new-deployment --region your-aws-region'
              }
            }
          }
        }
        post {
          always {
            sh 'docker logout'
          }
        }
      }
  ```

### Тестування та Розгортання

1.  Тестуйте Pipeline (фаза тестів була прописана в jenkinsfile):

    -   Здійсніть зміни у вашому репозиторії, щоб викликати pipeline.
    -   Перевірте статус та логи збірки в Jenkins.
      
2.  Налаштування Webhooks (також описано в туторіалі 2 і виконується з допомогою github):

    -   Конфігуруйте webhook у вашому Git репозиторії для автоматичного виклику Jenkins job при кожному push у головну гілку.+

Налаштування Сповіщень
----------------------

### Налаштування Сповіщень Електронною Поштою

1.  Налаштуйте Jenkins для відправки електронних листів:

    -   Відкрийте Jenkins Dashboard і перейдіть до 'Manage Jenkins' > 'Configure System'.
    -   Знайдіть розділ 'E-mail Notification'.
    -   Введіть інформацію SMTP сервера вашого електронного поштового сервісу. Наприклад, для Gmail це буде:
        -   SMTP server: `smtp.gmail.com`
        -   Use SSL: Yes
        -   SMTP Port: `465`
    -   Введіть вашу електронну адресу та пароль (може знадобитися створити спеціальний пароль для програм, якщо ви використовуєте двофакторну автентифікацію).
    -   Натисніть 'Test configuration by sending test e-mail' для перевірки налаштувань.
    -   Збережіть налаштування.
2.  Додайте сповіщення до вашого Jenkinsfile:

    -   У  `Jenkinsfile`, додайте наступний код у відповідну стадію:

        ```groovy

        post {
            success {
                mail to: 'your-email@example.com',
                     subject: "Build Successful: ${currentBuild.fullDisplayName}",
                     body: "The build was successful."
            }
            failure {
                mail to: 'your-email@example.com',
                     subject: "Build Failed: ${currentBuild.fullDisplayName}",
                     body: "The build failed."
            }
        }
        ```

### Інтеграція з Telegram

1.  Створіть Telegram Bot:

    -   Напишіть @BotFather у Telegram.
    -   Виконайте команду `/newbot` та слідуйте інструкціям для створення нового бота.
    -   Запишіть токен, який вам надасть BotFather.
2.  Встановіть та налаштуйте Jenkins Telegram Plugin:

    -   Перейдіть до 'Manage Jenkins' > 'Manage Plugins'.
    -   Знайдіть та встановіть 'Telegram Notifications' плагін.
    -   Після встановлення перейдіть до 'Manage Jenkins' > 'Configure System'.
    -   Знайдіть налаштування 'Telegram' і введіть ваш токен бота.
3.  Додайте сповіщення Telegram до вашого Jenkinsfile:

    -   У вашому `Jenkinsfile`, додайте наступний код у відповідну стадію:

        ```groovy

        post {
            success {
                telegramSend "Build Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            }
            failure {
                telegramSend "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            }
        }
     
