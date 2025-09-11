**Syfte:** Bygga en robust och skalbar WordPress-miljö på AWS med ALB + ASG (EC2 med WordPress-AMI), RDS (databas) och EFS (media), skyddat med Security Groups.

Min approach utgick från att skapa en lösning som är robust, skalbar och följer best practice inom molninfrastruktur. Istället för att köra hela WordPress-miljön på en enda EC2-instans (en klassisk LAMP-stack), valde jag att separera komponenterna: databasen i en hanterad RDS-tjänst, media i EFS och själva WordPress-koden i stateless EC2-instanser bakom en ALB. På så sätt kan varje del skötas, skala och säkras på sitt håll.

Målet var inte bara att få igång WordPress, utan att bygga en grund som kan växa — där det är enkelt att byta ut eller skala upp enskilda delar utan att störa helheten. Lösningen jag byggt är enklare än en full produktionsmiljö, men följer samma principer och kan byggas vidare med t.ex. WAF, HTTPS och CI/CD i framtiden.
