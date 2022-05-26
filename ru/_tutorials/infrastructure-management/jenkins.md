# Автоматизация сборки образов с помощью Jenkins и Packer

На основе заданной конфигурации Packer создает [образы дисков ВМ](../../compute/concepts/image.md) в сервисе {{ compute-name }}. Jenkins позволяет построить процесс непрерывной доставки изменений.

Образы можно использовать при создании облачной инфраструктуры, например, с помощью [Terraform](https://www.terraform.io/language#about-the-terraform-language).

Чтобы установить и настроить Jenkins, Packer, GitHub и Terraform для совместной работы:

1. [Подготовьте облако к работе](#before-you-begin).
1. [Настройте окружение](#prepare).
1. [Создайте сервисный аккаунт](#create-service-account).
1. [Создайте виртуальную машину с Jenkins](#create-jenkins-vm).
1. [Установите Packer на ВМ](#install-packer).
1. [Настройте Jenkins](#configure-jenkins).
1. [Настройте задачу для Jenkins](#jenkins-job).
1. [Настройте GitHub-репозиторий](#configure-github-repo).
1. [Создайте образ с помощью Jenkins](#create-image).
1. [Разверните образы с помощью Terraform](#deploy-image).

Если созданные ВМ и образы больше не нужны, [удалите их](#clear-out).

## Подготовьте облако к работе {#before-you-begin}

Перед тем, как разворачивать приложения, нужно зарегистрироваться в {{ yandex-cloud }} и создать платежный аккаунт:

{% include [prepare-register-billing](../_common/prepare-register-billing.md) %}

Если у вас есть активный платежный аккаунт, вы можете создать или выбрать каталог, в котором будет работать ваша виртуальная машина, на [странице облака](https://console.cloud.yandex.ru/cloud).
 
 [Подробнее об облаках и каталогах](../../resource-manager/concepts/resources-hierarchy.md).

### Необходимые платные ресурсы {#paid-resources}

В стоимость поддержки инфраструктуры входят:

* плата за постоянно запущенные виртуальные машины (см. [тарифы {{ compute-full-name }}](../../compute/pricing.md));
* плата за хранение созданных образов (см. [тарифы {{ compute-full-name }}](../../compute/pricing#prices-storage));
* плата за использование динамических публичных IP-адресов (см. [тарифы {{ vpc-full-name }}](../../vpc/pricing.md)).

## Настройте окружение {#prepare}

Подготовьте программы для работы:

* [Установите](../../cli/operations/install-cli.md) интерфейс командной строки {{ yandex-cloud }}.
* [Установите](https://www.terraform.io/downloads) Terraform. См. также раздел [{#T}](terraform-quickstart.md).
* [Загрузите](https://stedolan.github.io/jq/download/) утилиту jq.
* [Настройте](https://gitforwindows.org) Git. Если вы работаете под Windows, используйте Git Bash.
* [Создайте](https://github.com/yandex-cloud/examples) ответвление репозитория с примерами в своем аккаунте на GitHub.
* [Подготовьте](../../compute/operations/vm-connect/ssh.md) SSH-ключ для доступа к виртуальным машинам.

## Создайте сервисный аккаунт {#create-service-account}

С помощью сервисного аккаунта Jenkins сможет выполнять действия в вашем облаке и каталоге. Чтобы создать сервисный аккаунт:

1. Получите идентификаторы каталога и облака, выполнив команду `yc config list`.
1. Создайте сервисный аккаунт и передайте его идентификатор в переменную окружения, выполнив команды:

   ```
   yc iam service-account create --name <имя пользователя>
   yc iam key create --service-account-name <имя пользователя> -o <имя пользователя.json>
   SERVICE_ACCOUNT_ID=$(yc iam service-account get --name <имя пользователя> --format json | jq -r .id)
   ```

   В текущем каталоге будет создан JSON-файл, содержащий авторизационные данные.

1. Назначьте сервисному аккаунту роль `admin` на каталог, где будут выполняться операции:

   ```
   yc resource-manager folder add-access-binding <имя_каталога> --role admin --subject serviceAccount:$SERVICE_ACCOUNT_ID
   ```

## Создайте виртуальную машину с Jenkins {#create-jenkins-vm}

Jenkins будет получать изменения в конфигурациях образов ВМ из GitHub, а затем с помощью Packer создавать образы в облаке.

Чтобы создать виртуальную машину с Jenkins:

1. На странице каталога в [консоли управления]({{ link-console-main }}) нажмите кнопку **Создать ресурс** и выберите **Виртуальная машина**.
1. В поле **Имя** введите имя виртуальной машины: `jenkins-tutorial`.
1. Выберите [зону доступности](../../overview/concepts/geo-scope.md), в которой будет находиться виртуальная машина.
1. В блоке **Выбор образа/загрузочного диска** перейдите на вкладку **{{ marketplace-name }}** и нажмите кнопку **Посмотреть больше**. В открывшемся окне выберите образ [Jenkins](https://cloud.yandex.ru/marketplace/products/yc/jenkins-2-204-2).

    {% note info %}

    В случае самостоятельной настройки ВМ с Jenkins воспользуйтесь [инструкцией](https://www.jenkins.io/doc/book/installing/linux/).

    {% endnote %}

1. В блоке **Диски** укажите размер загрузочного диска 15 ГБ.
1. В блоке **Вычислительные ресурсы**:
    - Выберите [платформу](../../compute/concepts/vm-platforms.md): Intel Ice Lake.
    - Укажите необходимое количество vCPU и объем RAM:
       * **vCPU** — 2.
       * **Гарантированная доля vCPU** — 20%.
       * **RAM** — 2 ГБ.

1. В блоке **Сетевые настройки** нажмите кнопку **Добавить сеть** и выберите, к какой подсети подключить виртуальную машину. В блоке **Публичный адрес** назначьте ВМ публичный адрес автоматически или выберите один из зарезервированных адресов. 
1. В блоке **Доступ** укажите данные для доступа на виртуальную машину:
    - В поле **Логин** введите имя пользователя.
    - В поле **SSH-ключ** вставьте содержимое файла открытого ключа.
        Пару ключей для подключения по SSH необходимо [создать](../../compute/operations/vm-connect/ssh.md#creating-ssh-keys) самостоятельно.
1. Нажмите кнопку **Создать ВМ**.

## Установите Packer {#install-packer}

Packer позволяет создавать [образы дисков виртуальных машин](../../compute/concepts/image.md) с заданными в конфигурационном файле параметрами.

{% note info %}

Для работы с {{ yandex-cloud }} требуется Packer версии не ниже 1.5.

{% endnote %}

1. Скачайте дистрибутив [Packer](https://packer.io/downloads) для Linux.
1. Загрузите Packer на созданную ВМ: 

   ```
   scp packer_<версия Packer>_linux_amd64.zip <Логин>@<Публичный IP-адрес ВМ>:~/
   ```

1. [Подключитесь](../../compute/operations/vm-connect/ssh.md) к виртуальной машине по протоколу SSH. Для этого можно использовать утилиту `ssh` в Linux и macOS и программу `PuTTY` для Windows.
1. Создайте новую директорию, переместите в нее исполняемые файлы Packer и распакуйте архив:

   ```
   sudo mkdir /opt/yandex-packer/
   sudo mv packer_<версия Packer>_linux_amd64.zip /opt/yandex-packer/
   unzip packer_<версия Packer>_linux_amd64.zip
   ```
1. Все действия системы Jenkins будут выполняться от имени пользователя `jenkins`. Дайте этому пользователю права на запуск Packer:

   ```
   sudo chmod u+x /opt/yandex-packer/packer*
   sudo chown jenkins:jenkins /opt/yandex-packer/packer*
   ```

## Настройте Jenkins {#configure-jenkins}

Чтобы собирать образы по конфигурациям из GitHub, нужно настроить Jenkins:

1. [Подключитесь](../../compute/operations/vm-connect/ssh.md) к виртуальной машине по протоколу SSH. Для этого можно использовать утилиту `ssh` в Linux и macOS и программу `PuTTY` для Windows.
1. Откройте файл пароля для запуска настройки и скопируйте пароль:

   ```
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
1. Перейдите в браузере по адресу `http://<публичный IP-адрес ВМ с Jenkins>`. Откроется консоль управления Jenkins.
1. Введите в поле **Administrator password** скопированный пароль и нажмите кнопку **Continue**.
1. Выберите **Select plugins to install**.

   Вам потребуются следующие плагины:

   * `Pipeline` — плагин для получения исходного кода из системы контроля версий, его сборки, тестирования и развертывания. 
   * `Git` — плагин для работы с Git-репозиториями.
   * `Credentials Binding` — плагин для создания переменных окружения, содержащих авторизационные данные.

1. Нажмите кнопку **Install**. Начнется установка выбранных компонентов.
1. После завершения установки вам будет предложено создать учетную запись администратора. Заполните поля формы и нажмите кнопку **Save and Continue**.
1. Вам будет предложено создать URL для Jenkins. Оставьте URL вида `http://<публичный IP-адрес ВМ>/`. Нажмите кнопку **Save and finish**.
1. Нажмите кнопку **Start using Jenkins**, чтобы завершить установку и перейти на административную панель Jenkins.

## Настройте задачу для Jenkins {#jenkins-job}

Чтобы Jenkins мог выполнять сборки образов, следует указать авторизационные данные для {{ yandex-cloud }} и создать задачу на получение изменений из репозитория GitHub. Авторизационные данные будут использоваться в переменных, находящихся в конфигурационных файлах Packer.

1. Откройте административную панель Jenkins.
1. В правом верхнем углу нажмите на имя пользователя.
1. Выберите пункт **Credentials**.
1. В блоке **Stores scoped to Jenkins** нажмите на ссылку `Global`.
1. Получите идентификатор подсети, в которой будут собираться образы, выполнив команду `yc vpc subnet list`.
1. Нажмите кнопку **Add credentials**. Укажите следующие параметры:

   1. В списке **Kind** выберите пункт `Secret text`.
   1. В списке **Scope** оставьте `Global`.
   1. В поле **Secret** укажите идентификатор вашего каталога.
   1. В поле **Id** укажите `YC_FOLDER_ID`. Нажмите кнопку **OK**.

1. Создайте еще один секрет со следующими параметрами:

   1. **Kind**: `Secret text`.
   1. **Scope**: `Global`.
   1. **Secret**: идентификатор подсети, в которой находится ВМ с Jenkins.
   1. **ID**: `YC_SUBNET_ID`.

1. Создайте еще один секрет со следующими параметрами:

   1. **Kind**: `Secret file`.
   1. **Scope**: `Global`.
   1. **File**: файл `<имя пользователя>.json` из [шага 1](#create-service-account).
   1. **ID**: `YC_ACCOUNT_KEY_FILE`.

1. Вернитесь на главную страницу административной панели и выберите пункт **New item**.
1. Введите название для задачи: `jenkins-tutorial` и выберите тип задачи **Pipeline**. Нажмите кнопку **OK**.
1. В открывшемся окне поставьте флаг **GitHub hook trigger for GITScm polling**. Эта опция позволяет запускать сборку по каждому выполнению команды `push` в ветку `master` Git-репозитория.
1. В блоке **Pipeline** в списке **Definition** выберите `Pipeline script from SCM`.
1. В списке **SCM** выберите `Git`.
1. В поле **Repository URL** укажите URL вашего ответвления из GitHub.
1. В поле **Script path** укажите `jenkins-packer/Jenkinsfile`.
1. Оставьте остальные поля без изменений и нажмите **Сохранить**.

## Настройте GitHub-репозиторий {#configure-github-repo}

В настройках репозитория GitHub включите webhook для запуска сборки в Jenkins и добавьте публичный SSH-ключ для авторизации.

### Включите Webhook {#configure-webhook}

1. Откройте ответвление репозитория на GitHub в браузере.
1. Выберите вкладку **Settings**.
1. Выберите пункт **Webhooks** и нажмите кнопку **Add webhook**.
1. В поле **Payload URL** введите `http://<публичный IP-адрес ВМ>/github-webhook/`.
1. Нажмите кнопку **Add webhook**.

### Добавьте на GitHub SSH-ключ {#add-ssh-key}

1. Нажмите на ваш аватар на GitHub. В открывшемся меню выберите пункт **Settings**.
1. Выберите пункт **SSH and GPG keys**.
1. Нажмите кнопку **New SSH key**.
1. В поле **Title** введите любое имя ключа.
1. Скопируйте в поле **Key** ваш SSH-ключ.
1. Нажмите кнопку **Add SSH key**.

## Создайте образ с помощью Jenkins {#create-image}

Сборка образа в Jenkins запускается автоматически после выполнения команды `push` в ветке `master` GitHub-репозитория. 

1. Склонируйте на ваш компьютер ответвление репозитория [examples](https://github.com/yandex-cloud/examples), которое вы создали во время [подготовки к работе](#before-you-begin):
   
   ```
   git clone git@github.com:<логин на GitHub>/examples.git
   ```

1. Внесите изменения в шаблоны Packer, находящиеся в директории `jenkins-packer/packer/`. Документацию шаблонов Packer можно найти на [сайте](http://packer.io/docs/templates/index.html) разработчика. В параметрах `image_family` и `source_image_family` указываются семейства образов, которые будет собирать Jenkins. Подробнее о семействах см. [Семейства образов](../../compute/concepts/image#family).
1. Внесите изменения в файл описания Pipeline для Jenkins `Jenkinsfile`, расположенный в корневой директории репозитория. Документацию Pipeline см. на [сайте](https://jenkins.io/doc/book/pipeline/syntax/) разработчика. 
1. Загрузите изменения на GitHub:

   ```
   git add -A
   git commit -m "Build update"
   git push
   ```

1. Откройте административную панель Jenkins и проверьте состояние задачи.
1. Если все настройки выполнены верно, то запустится сборка образов. Результат выполнения можно увидеть в логах сборки. 

{% note info %}

При настройке задачи Jenkins в разделе **GitHub Hook log** возможно появление ошибки `Polling has not run yet`. В этом случае следует первый раз запустить сборку вручную.

{% endnote %}

После этого в разделе **Образы** сервиса **Compute Cloud** появятся три новых образа: 

* `Debian` — базовый образ с последними обновлениями. 
* `Nginx` — образ с веб-сервером nginx, базирующийся на образе `Debian`.
* `Django` — образ с фреймворком Django, базирующийся на образе `Debian`.

## Разверните образы {#deploy-image}

После того, как образы будут созданы, их можно использовать для создания виртуальных машин. Создайте тестовую инфраструктуру с помощью Terraform:

1. В директории с ответвлением перейдите в директорию с файлами Terraform:

   ```
   cd examples/jenkins-packer/terraform
   ```

1. Переименуйте файл `terraform.tfvars_example`:

   ```
   mv terraform.tfvars_example terraform.tfvars
   ```

1. Заполните поля файла требуемыми значениями. См. также документацию [Terraform](https://www.terraform.io/language#about-the-terraform-language) и [провайдера {{ yandex-cloud }}](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs).
1. Инициализируйте провайдера Terraform командой `terraform init`.
1. Выполните команду `terraform plan -var-file="terraform.tfvars"`. Проверьте созданную конфигурацию.
1. Выполните команду `terraform apply` и подтвердите создание инфраструктуры, введя `yes` в терминале.

После этого будут созданы: 

1. Облачная сеть. 
1. Подсети во всех зонах доступности. 
1. Виртуальные машины из образов, созданных с помощью Packer. Виртуальные машины с nginx получат публичные IP-адреса. Все виртуальные машины будут подключены к подсетям.

## Как удалить созданные ресурсы {#clear-out}

Чтобы освободить ресурсы в каталоге:

* [Удалите созданные ВМ](../../compute/operations/vm-control/vm-delete.md).
* [Удалите созданные образы](../../compute/operations/image-control/delete.md).
* [Удалите сервисный аккаунт](../../iam/operations/sa/delete.md) и файл `<имя пользователя.json>`.
* [Удалите сеть и подсеть](../../vpc/operations/network-delete.md).

Для удаления созданных с помощью Terraform используйте команду `terraform destroy`.