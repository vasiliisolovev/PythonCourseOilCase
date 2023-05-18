<h1 style="text-align: center;">  Пошаговая инструкция контейнеризации и развертывания OilCase </h1>

## 1. Dockerfile

1.1. MS Studio сама генерирует докерфайл решения, но его нужно несколько доработать. Так выглядит конечный вариант докерфайла:
```
# docker build -t ubuntu/oilcase --build-arg IP_ADDRESS="$(hostname -i)" .
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 1433 
ARG IP_ADDRESS
ENV HOST_IP=$IP_ADDRESS

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["OilCaseApi.csproj", "."]
RUN dotnet restore "./OilCaseApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "OilCaseApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "OilCaseApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
COPY Files ./Files

ENV ASPNETCORE_ENVIRONMENT Development

ENTRYPOINT ["dotnet", "OilCaseApi.dll", "--environment=Development"]
```

было добавлено выделение 8080 порта для самого приложения и 1433 порта для SQLServer. Выделение built-time переменной `IP_ADDRESS` нужно для того, чтобы передать в нее значение ip-адреса хоста во время сборки. Это значение сохраняется в переменную контейнера `HOST_IP` и используется при создании подключения с SQLServer в `ProgramOilcase.cs`. Также значение `ASPNETCORE_ENVIRONMENT` было изменено со `Staging` на `Development`.

1.2. На сервер OilCase можно перенести как целую директорию (используя, например, `WINSCP`) и собрать образ уже на сервере. Это удобно потому, что можно делать поправки в коде прямо на сервере. Или же можно каждый раз загружать туда измененный докер-образ.

1.3. Выбрав первый вариант, переходим в директорию внутри OilCase, в которой лежит докерфайл. Выполняем сборку:
```
 docker build -t ubuntu/oilcase --build-arg IP_ADDRESS="$(hostname -i)" .
```
1.4. После чего запускаем контейнер, не забывая пробросить использующиеся порты:
```
docker run -p 8080:8080 ubuntu/oilcase
```


\* В файлах проекта были для контейнеризации сделаны следующие изменения:
1) Строка подключения поменялась с 
`ApplicationContext.ConnectionString = _builder.Configuration.GetConnectionString(_connectionStringName);` 
на 
`ApplicationContext.ConnectionString = $"Data Source={Environment.GetEnvironmentVariable("HOST_IP")}, 1433;Initial Catalog=OILCASEDB;Persist Security Info=True; MultipleActiveResultSets=true; User ID=oilcase;Password=******";` 

2) URL ОйлКейса для соответсвия конвенциям убунты из `http://localhost:8080` стал `http://0.0.0.0:8080`