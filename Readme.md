# dbops-project

Исходный репозиторий для выполнения проекта дисциплины "DBOps"

1. CREATE DATABASE "store";
2. CREATE ROLE migration_service_user WITH LOGIN PASSWORD 'super_secret';
3. GRANT ALL PRIVILEGES ON DATABASE store TO migration_service_user;
4. GRANT ALL ON SCHEMA public TO migration_service_user;
5. GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO migration_service_user;
6. ALTER DEFAULT PRIVILEGES FOR USER migration_service_user IN SCHEMA public
   GRANT ALL PRIVILEGES ON TABLES TO migration_service_user;
