# GitLab_local
Настройка собственного GitLab CI/CD сервера с помощью Docker Compose

# Введение
Сегодня я расскажу вам как развернуть собственный локальный GitLab сервер со всеми необходимыми сервисами (dind, gitlab-runners, register-runner) с использованием Docker Compose.
Этот гайд поможет вам создать локальную среду для практики с CI/CD процессами. Мы пройдем через все этапы: от настройки контейнеров до регистрации раннеров и создания примера CI/CD пайплайна.  

# Зачем нужен собственный GitLab сервер?
### Собственный GitLab сервер позволяет:  
### 1. Иметь полный контроль над данными и безопасностью  
 - Вы можете хранить весь исходный код, конфиденциальные данные и CI/CD пайплайны локально,  
   без необходимости передавать их в облачные сервисы.  
### 2. Гибкая настройка под свои нужды  
 - Собственный GitLab сервер можно настроить под конкретные требования вашей команды или компании:  
   интеграции с внутренними системами, кастомизация рабочих процессов, добавление собственных плагинов и скриптов.  
### 3. Создать тестовую площадку для отработки CI/CD практик  
 - Локальный GitLab сервер идеально подходит для обучения и экспериментов с CI/CD пайплайнами, DevOps-практиками  
   и автоматизацией процессов разработки. Вы можете тестировать новые подходы без риска повлиять на реальные проекты.  

# Как запустить свой сервер GitLab в контейнере?  

### 1. Подготовка файловой структуры:  
```
mkdir gitlab-local
cd gitlab-local

# Создайте директории для хранения данных
mkdir -p ./data/docker/gitlab/{var/opt/gitlab,var/log/gitlab,etc/gitlab-runner,var/run/docker.sock}
mkdir -p ./config
touch ./config/config.toml
```
### 2. Запуск Docker Compose:  
- Создайте файл ```docker-compose.yml``` в директории ```gitlab-local```  
- Скопируйте содержимое файла к себе в .yaml  
 [Ссылка на docker-compose.yml](https://github.com/MichailFedyaev/GitLab_local/blob/main/docker-compose.yml)  
 ```nano docker-compose.yml```  
- Теперь запустите контейнеры:  
 ```docker-compose up -d```  
- После чего проверьте логи контейнера dind  
 ```docker-compose logs dind```  
 Если вы видите ошибку:  
 ```
 failed to start daemon: error initializing graphdriver:  
 error changing permissions on file for metacopy check: permission denied: overlay2
 ```  
 Это связано с тем, что директории создаются на файловой системе Windows (NTFS), которая не поддерживает UNIX права.  
 ### Временно решение:  
 - Используйте драйвер ```vfs``` вместо ```overlay2```:  
```python  
command: [  
"--host", "tcp://0.0.0.0:2375",  
"--storage-driver=vfs",  
"--tls=false"  
]
```  
```vfs``` менее производительный чем ```overlay2```, но и менее требовательный  
Поэтому если возникает данная ошибка то на стадии настройки лучше поставить именно его.   
### Полноценное решение:  
 - Переместите данные в файловую систему Linux внутри WSL.  
### 3. Доступ к веб-интерфейсу GitLab:  
- URL: ```http://localhost:8000/```
- Логин: ```root```
- Пароль: ```CHANGEME123```

### 4. Настройка SSH-ключа:
- Перейдите в настройки пользователя GitLab  
   URL: ```http://localhost:8000/-/user_settings/ssh_keys```  
- Добавьте новый SSH-ключ  

# Регистрация и настройка GitLab Runner для выполнения задач в Docker

### 1. Создайте и заполните файл .env 
```
nano .env  

RUNNER_NAME=ИмяПроекта  
REGISTRATION_TOKEN=ВашТокенРегистрации  
```
### 2. Теперь нужно получить REGISTRATION_TOKEN:
- Создайте новый проект (```test```)  
- Зайдите в Settings → CI/CD → Runners (```http://localhost:8000/root/test/-/settings/ci_cd```)  
- Скопируйте свой Registration Token  
  ![image](https://github.com/user-attachments/assets/d7da7a74-0be7-49a9-a6d1-a8f56d5994fd)  
- Обновите файл .env с новым токеном и именем раннера (```test```).

### 3. Перезапустите сервис:  
```docker-compose restart register-runner```  
После регистрации раннера перейдите по URL: ```http://localhost:8000/admin/runners```  
Вы должны увидеть похожую картину  
![image](https://github.com/user-attachments/assets/9e4d2e2b-c57c-4e78-b01b-39dbbe9bac22)

# Пример настройки проекта с GitLab CI/CD  
1. Клонируйте проект который создали ранее (```test```)  
  ```git clone http://localhost:8000/root/test.git```  
  (или по SSH ```git clone ssh://git@localhost:8822/root/test.git```)  
2. Создаем ветку и файлы  
  ```git checkout -b test_ci```  
3. Создаем файл main.py  
  ```python
  def main():
    """
    This function prints "Hello World" to the console.
    """
    print('Hello World')

  main()
  ```  
4. Создаём файл .gitlab-ci.yml  
   Копируем содержимое файла к себе в .yaml  
  [Ссылка на .gitlab-ci.yml](https://github.com/MichailFedyaev/GitLab_local/blob/main/.gitlab-ci.yml)  
  ```nano .gitlab-ci.yml.yml```  
6. Фиксируем изменения ```git add .```  
7. Комитим изменения ```git commit -m "feat: #12345 Добавлен CI пайплайн"```  
8. Отправляем изменения ```git push -u origin test_ci```  
9. После чего создаем Merge Request  
  Открываем созданный проект ```test```
  Нажмите "Create merge request" для ветки ```test_ci```
10. Проверьте выполнение пайплайна на вкладке "Сборочная линия"  
  В проекте: ```CI/CD``` → ```Pipelines```  
  Должны отображаться два задания: ```commit-name-test``` и ```pylint```  

### Этот гайд поможет вам настроить локальную среду GitLab CI/CD для изучения и экспериментов. Помните, что для продакшен-среды потребуются дополнительные настройки безопасности и производительности.
