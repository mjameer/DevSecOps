# Created this project to Implement DevSecOps

<img width="807" alt="image" src="https://github.com/mjameer/Ekart/assets/11364104/d0a9ad69-d277-4d9f-a6f3-da9cb70bd2ac">

This is a demo project made using Spring + Thymeleaf, **Spring Boot**, **Spring Security**, **Thymeleaf**, **Spring Data JPA**, **Spring Data REST and Docker**. 
Database is in memory **H2**.

## How to run

You can run it via Maven or Docker. 

Once the app starts, go to the web browser and visit `http://localhost:8070/home`

```
ID/PWD: **admin**/**admin**
ID/PWD: **user**/**password**
```
## H2 Database web interface

Go to the web browser and visit `http://localhost:8070/h2-console`

In field **JDBC URL** put 
```
jdbc:h2:mem:shopping_cart_db
```
