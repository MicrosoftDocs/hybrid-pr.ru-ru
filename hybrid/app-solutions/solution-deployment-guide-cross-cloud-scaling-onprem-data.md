---
title: Развертывание гибридного приложения с локальными данными, масштабируемого в разных облаках
description: Узнайте, как развернуть приложение, использующее локальные данные и выполняющее масштабирование в нескольких облаках с помощью Azure и Azure Stack Hub.
author: BryanLa
ms.topic: article
ms.date: 11/05/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 11/05/2019
ms.openlocfilehash: 0989859fd68847932d3e69defee59740a2bffd44
ms.sourcegitcommit: 962334135b63ac99c715e7bc8fb9282648ba63c9
ms.translationtype: HT
ms.contentlocale: ru-RU
ms.lasthandoff: 03/23/2021
ms.locfileid: "104895403"
---
# <a name="deploy-hybrid-app-with-on-premises-data-that-scales-cross-cloud"></a>Развертывание гибридного приложения с локальными данными, масштабируемого в разных облаках

В этом руководстве по решению показано, как развернуть гибридное приложение, которое охватывает Azure и Azure Stack Hub и использует один источник локальных данных.

Используя гибридное облачное решение, можно объединить преимущества соответствия частного облака с масштабируемостью общедоступного. Ваши разработчики могут воспользоваться преимуществами экосистемы разработчиков Майкрософт и применить свои навыки работы в облаке и локальных средах.

## <a name="overview-and-assumptions"></a>Обзор и предположения

В этом руководстве описано, как настроить рабочий процесс, который позволяет разработчикам развернуть веб-приложение в общедоступном и частном облаках. Это приложение сможет получить доступ к маршрутизируемой сети, недоступной из Интернета и размещенной в частном облаке. Такие веб-приложения отслеживаются, и при резком увеличении трафика программа изменяет записи DNS, чтобы перенаправить трафик в общедоступное облако. Когда трафик опускается до уровня, предшествующего пику, он направляется обратно в частное облако.

В рамках этого руководства рассматриваются следующие задачи:

> [!div class="checklist"]
> - Развертывание сервера базы данных SQL Server, подключенного с помощью гибридного подключения.
> - Подключение веб-приложения в глобальной среде Azure к гибридной сети.
> - Настройка DNS для масштабирования в нескольких облаках.
> - Настройка SSL-сертификатов для масштабирования в нескольких облаках.
> - Настройка и развертывание веб-приложения.
> - Создание и настройка профиля диспетчера трафика для масштабирования в нескольких облаках.
> - Настройка мониторинга и оповещений Application Insights для большего объема трафика.
> - Настройка автоматического переключения трафика между глобальной средой Azure и Azure Stack Hub.

> [!Tip]  
> ![Схема основных аспектов проектирования гибридных приложений](./media/solution-deployment-guide-cross-cloud-scaling/hybrid-pillars.png)  
> Microsoft Azure Stack Hub — это расширение Azure. Azure Stack Hub обеспечивает гибкость и высокую скорость внедрения инноваций облачных вычислений в локальной среде. Это решение позволяет использовать единственное гибридное облако, с помощью которого можно создавать и развертывать гибридные приложения в любой точке мира.  
> 
> В руководстве по [проектированию гибридных приложений](overview-app-design-considerations.md) перечислены основные аспекты качественного программного обеспечения (размещение, масштабируемость, доступность, устойчивость, управляемость и безопасность), которые следует учитывать при разработке, развертывании и использовании гибридных приложений. Эти рекомендации помогут оптимизировать разработку гибридных приложений и предотвратить появление проблем с рабочими средами.

### <a name="assumptions"></a>Предположения

В этом учебнике также предполагается, что вы знакомы с глобальной средой Azure и Azure Stack Hub. Если вы хотите узнать больше, прежде чем начать работу с руководством, прочтите следующие статьи:

- [Введение в Azure](https://azure.microsoft.com/overview/what-is-azure/).
- [Обзор Azure Stack Hub](/azure-stack/operator/azure-stack-overview)

В этом руководстве также предполагается, что у вас есть подписка Azure. Если у вас еще нет подписки, [создайте бесплатную учетную запись Azure](https://azure.microsoft.com/free/), прежде чем начинать работу.

## <a name="prerequisites"></a>Предварительные требования

Перед началом работы с этим решением убедитесь, что выполнены следующие требования.

- Пакет средств разработки Azure Stack (ASDK) или подписка на интегрированную систему Azure Stack Hub. Чтобы развернуть ASDK, следуйте инструкциям в разделе [Развертывание пакета SDK для Azure Stack](/azure-stack/asdk/asdk-install).
- В вашей установке Azure Stack Hub должны быть установлены следующие компоненты:
  - Служба приложений Azure. Обратитесь к своему оператору Azure Stack Hub, чтобы развернуть и настроить Службу приложений Azure в своей среде. Для работы с этим руководством требуется, чтобы в Службе приложений была доступна по крайней мере одна (1) выделенная рабочая роль.
  - Образ Windows Server 2016.
  - Windows Server 2016 с образом Microsoft SQL Server.
  - Соответствующие планы и предложения.
  - Доменное имя для веб-приложения. Если у вас нет доменного имени, его можно приобрести у поставщика доменов, таких как GoDaddy, Bluehost и InMotion.
- SSL-сертификат для вашего домена из доверенного центра сертификации, например LetsEncrypt.
- Веб-приложение, которое обменивается данными с Базой данных SQL Server и поддерживает Application Insights. Вы можете загрузить пример приложения [dotnetcore-sqldb-tutorial](https://github.com/Azure-Samples/dotnetcore-sqldb-tutorial) с GitHub.
- Гибридная сеть между виртуальной сетью Azure и виртуальной сетью Azure Stack Hub. Дополнительные инструкции см. в статье [Configure hybrid cloud connectivity with Azure and Azure Stack Hub](solution-deployment-guide-connectivity.md) (Настройка подключения к гибридному облаку с помощью Azure и Azure Stack Hub).

- Гибридный конвейер непрерывной интеграции и развертывания (CI/CD) с частным агентом сборки в Azure Stack Hub. Подробные сведения см. в статье о [настройке идентификатора гибридного облака для приложений Azure и Azure Stack Hub](solution-deployment-guide-identity.md).

## <a name="deploy-a-hybrid-connected-sql-server-database-server"></a>Развертывание базы данных SQL Server, подключенной с помощью гибридного подключения

1. Войдите на портал пользователя Azure Stack Hub.

2. На **панели мониторинга** выберите **Marketplace**.

    ![Azure Stack Hub Marketplace](media/solution-deployment-guide-hybrid/image1.png)

3. В **Marketplace** выберите **Вычисления**, а затем выберите **Дополнительно**. В разделе **Дополнительно** выберите образ **Free SQL Server License: SQL Server 2017 Developer on Windows Server** (Бесплатная лицензия на SQL Server: SQL Server 2017 Developer на базе Windows Server 2016).

    ![Выбор образа виртуальной машины на пользовательском портале Azure Stack Hub](media/solution-deployment-guide-hybrid/image2.png)

4. В разделе **Free SQL Server License: SQL Server 2017 Developer on Windows Server** (Бесплатная лицензия на SQL Server: SQL Server 2017 Developer на базе Windows Server) выберите **Создать**.

5. Выберите **Основные сведения > Настройка базовых параметров**, укажите **имя** виртуальной машины, **имя пользователя** для сопоставления безопасности SQL Server и **пароль** для сопоставления безопасности.  В раскрывающемся списке **Подписка** выберите подписку для развертывания. Для параметра **Группа ресурсов** выберите значение **Choose existing** (Выбрать существующую) и поместите виртуальную машину в ту же группу ресурсов, что и веб-приложение Azure Stack Hub.

    ![Настройка основных параметров виртуальной машины на пользовательском портале Azure Stack Hub](media/solution-deployment-guide-hybrid/image3.png)

6. В разделе **Размер** нужно выбрать размер виртуальной машины. В этом руководстве мы рекомендуем A2_Standard или DS2_V2_Standard.

7. В разделе **Настройки > Настроить дополнительные возможности** настройте следующие параметры:

   - **Учетная запись хранения**. Создайте учетную запись, если необходимо.
   - **Виртуальная сеть.**

     > [!Important]  
     > Убедитесь, что виртуальная машина SQL Server развернута в той же виртуальной сети, что и VPN-шлюзы.

   - **Общедоступный IP-адрес**. Оставьте параметры по умолчанию.
   - **Группа безопасности сети**. (NSG). Создайте NSG.
   - **Расширения и мониторинг**. Оставьте параметры по умолчанию.
   - **Учетная запись хранения для диагностики**. Создайте учетную запись, если необходимо.
   - Нажмите кнопку **ОК**, чтобы сохранить конфигурацию.

     ![Настройка дополнительных функций виртуальной машины на пользовательском портале Azure Stack Hub](media/solution-deployment-guide-hybrid/image4.png)

8. В разделе **Настройки SQL Server** настройте следующие параметры:

   - Для параметра **Подключение SQL** выберите **Общедоступный (Интернет)** .
   - Для параметра **Порт** оставьте значение по умолчанию **1433**.
   - Для параметра **Проверка подлинности SQL** выберите значение **Включить**.

     > [!Note]  
     > При включении параметра проверки подлинности SQL он должен быть автоматически заполнен сведениями "SQLAdmin", настроенными в разделе **Основные сведения**.

   - Для остальных параметров оставьте значения по умолчанию. Щелкните **ОК**.

     ![Настройка параметров SQL Server на пользовательском портале Azure Stack Hub](media/solution-deployment-guide-hybrid/image5.png)

9. В разделе **Сводка** проверьте конфигурацию ВМ, а затем нажмите кнопку **ОК**, чтобы начать развертывание.

    ![Сводка конфигурации на пользовательском портале Azure Stack Hub](media/solution-deployment-guide-hybrid/image6.png)

10. Создание виртуальной машины может занять некоторое время. Просмотреть состояние виртуальных машин можно в разделе **Виртуальные машины**.

    ![Состояние виртуальных машин на пользовательском портале Azure Stack Hub](media/solution-deployment-guide-hybrid/image7.png)

## <a name="create-web-apps-in-azure-and-azure-stack-hub"></a>Создание веб-приложений в Azure и Azure Stack Hub

Служба приложений Azure упрощает запуск веб-приложения и управление им. Так как Azure Stack Hub согласуется с Azure, Службу приложений можно запускать в обеих средах. Вы будете использовать службу приложений для размещения приложения.

### <a name="create-web-apps"></a>Создание веб-приложений

1. Создайте веб-приложение в Azure, следуя инструкциям в разделе [Создание плана службы приложений](/azure/app-service/app-service-plan-manage#create-an-app-service-plan). Веб-приложение должно находиться в той же подписке и группе ресурсов, что и ваша гибридная сеть.

2. Повторите предыдущий шаг (1) в Azure Stack Hub.

### <a name="add-route-for-azure-stack-hub"></a>Добавление маршрута для Azure Stack Hub

Служба приложений в Azure Stack Hub должна поддерживать маршрутизацию из общедоступного сегмента Интернета, чтобы пользователи могли получить доступ к приложению. Если вы можете получить доступ к Azure Stack Hub из Интернета, запишите общедоступный IP-адрес или URL-адрес веб-приложения Azure Stack Hub.

Если вы используете ASDK, вы можете [настроить статическое сопоставление NAT](/azure-stack/operator/azure-stack-create-vpn-connection-one-node#configure-the-nat-vm-on-each-asdk-for-gateway-traversal), чтобы предоставить службу приложений за пределами виртуального окружения.

### <a name="connect-a-web-app-in-azure-to-a-hybrid-network"></a>Подключение веб-приложения в среде Azure к гибридной сети

Чтобы обеспечить подключение между веб-интерфейсом в Azure и Базой данных SQL Server в Azure Stack Hub, веб-приложение должно быть подключено к гибридной сети между Azure и Azure Stack Hub. Чтобы настроить подключение, необходимо сделать следующее:

- настроить подключение "точка — сеть";
- настроить веб-приложение.
- изменить шлюз локальной сети в Azure Stack Hub.

### <a name="configure-the-azure-virtual-network-for-point-to-site-connectivity"></a>Настройка подключения "точка — сеть" в виртуальной сети Azure

Шлюз виртуальной сети в гибридной сети на стороне Azure должен разрешить подключения "точка — сеть" для интеграции со Службой приложений Azure.

1. На портале Azure перейдите на страницу шлюза виртуальной сети. В разделе **Параметры** выберите **Конфигурация "точка-сеть"** .

    ![Параметр "точка — сеть" в настройках шлюза виртуальной сети Azure](media/solution-deployment-guide-hybrid/image8.png)

2. Выберите **Настроить сейчас**, чтобы настроить подключение "точка — сеть".

    ![Запуск конфигурации "точка — сеть" в настройках шлюза виртуальной сети Azure](media/solution-deployment-guide-hybrid/image9.png)

3. На странице конфигурации **Точка — сеть** в поле **Пул адресов** введите диапазон частных IP-адресов, который вы хотите использовать.

   > [!Note]  
   > Указанный диапазон не должен перекрывать другие диапазоны адресов, которые уже используются в подсетях в глобальной системе Azure или компонентах Azure Stack Hub гибридной сети.

   В разделе **Тип туннеля** снимите флажок **IKEv2 VPN**. Нажмите кнопку **Сохранить**, чтобы завершить настройку подключения "точка — сеть".

   ![Параметры "точка — сеть" в настройках шлюза виртуальной сети Azure](media/solution-deployment-guide-hybrid/image10.png)

### <a name="integrate-the-azure-app-service-app-with-the-hybrid-network"></a>Интеграция приложения Службы приложений Azure с гибридной сетью

1. Чтобы подключить приложение к виртуальной сети Azure, следуйте инструкциям по [включению интеграции с виртуальной сетью, требуемой шлюзом](/azure/app-service/web-sites-integrate-with-vnet#gateway-required-vnet-integration).

2. Перейдите к разделу **Параметры** для плана службы приложений, в котором размещено веб-приложение. В разделе **Параметры** выберите **Сеть**.

    ![Конфигурация сети для плана службы приложений](media/solution-deployment-guide-hybrid/image11.png)

3. В разделе **Интеграция виртуальной сети** выберите **Щелкните здесь для управления**.

    ![Управление интеграцией виртуальной сети для плана службы приложений](media/solution-deployment-guide-hybrid/image12.png)

4. Выберите виртуальную сеть, которую требуется настроить. В разделе **IP-АДРЕСА, ПЕРЕНАПРАВЛЕННЫЕ В ВИРТУАЛЬНУЮ СЕТЬ** введите диапазон IP-адресов для виртуальной сети Azure, виртуальной сети Azure Stack Hub и адресных пространств "точка — сеть". Нажмите кнопку **Сохранить**, чтобы проверить и сохранить настройки.

    ![Диапазоны IP-адресов для маршрутизации в интеграции виртуальной сети](media/solution-deployment-guide-hybrid/image13.png)

Дополнительные сведения об интеграции службы приложений с виртуальными сетями Azure см. в статье [Интеграция приложения с виртуальной сетью Azure](/azure/app-service/web-sites-integrate-with-vnet).

### <a name="configure-the-azure-stack-hub-virtual-network"></a>Настройка виртуальной сети Azure Stack Hub

Для шлюза локальной сети в виртуальной сети Azure Stack Hub необходимо настроить маршрутизацию трафика из диапазона адресов "точка — сеть" Службы приложений.

1. На портале Azure Stack Hub выберите **Шлюз локальной сети**. В разделе **Параметры** выберите пункт **Конфигурация**.

    ![Параметр конфигурации шлюза в настройках шлюза локальной сети Azure Stack Hub](media/solution-deployment-guide-hybrid/image14.png)

2. В поле **Адресное пространство** введите диапазон адресов для подключения "точка — сеть" для шлюза виртуальной сети в Azure.

    ![Адресное пространство "точка — сеть" в настройках шлюза локальной сети Azure Stack Hub](media/solution-deployment-guide-hybrid/image15.png)

3. Нажмите кнопку **Сохранить**, чтобы проверить и сохранить конфигурацию.

## <a name="configure-dns-for-cross-cloud-scaling"></a>Настройка DNS для масштабирования в нескольких облаках

Благодаря правильной настройке DNS для облачных приложений пользователи могут получать доступ к глобальной среде Azure и экземплярам Azure Stack Hub веб-приложения. Конфигурация DNS для этого руководства также позволяет диспетчеру трафика Azure перенаправлять трафик при изменении нагрузки.

Для управления Azure DNS в этом руководстве используется Azure DNS, иначе домены Службы приложений не будут работать.

### <a name="create-subdomains"></a>Создание дочерних доменов

Так как диспетчер трафика использует имена DNS CNAME, дочерний домен необходим для правильной маршрутизации трафика в конечные точки. Подробные сведения о сопоставлении записей DNS и доменных имен с использованием диспетчера трафика см. в [этой статье](/azure/app-service/web-sites-traffic-manager-custom-domain-name).

Для конечной точки Azure вы создадите дочерний домен, с помощью которого пользователи смогут получать доступ к веб-приложению. Для этого руководства можно использовать **app.northwind.com**, но это значение необходимо настроить на основе вашего собственного домена.

Кроме того, необходимо создать дочерний домен с записью A для конечной точки Azure Stack Hub. Вы можете использовать **azurestack.northwind.com**.

### <a name="configure-a-custom-domain-in-azure"></a>Настройка личного домена в Azure

1. Добавьте имя узла **app.northwind.com** в веб-приложении Azure путем [сопоставления записи CNAME со Службой приложений Azure](/azure/app-service/app-service-web-tutorial-custom-domain#map-a-cname-record).

### <a name="configure-custom-domains-in-azure-stack-hub"></a>Настройка личных доменов в Azure Stack Hub

1. Добавьте имя узла **azurestack.northwind.com** в веб-приложении Azure Stack Hub путем [сопоставления записи A со Службой приложений Azure](/azure/app-service/app-service-web-tutorial-custom-domain#map-an-a-record). Используйте маршрутизируемый через Интернет IP-адрес для приложения Службы приложений.

2. Добавьте имя узла **app.northwind.com** в веб-приложении Azure Stack Hub путем [сопоставления записи CNAME со Службой приложений Azure](/azure/app-service/app-service-web-tutorial-custom-domain#map-a-cname-record). Используйте имя узла, которое вы настроили на предыдущем шаге (1), как целевой объект для записи CNAME.

## <a name="configure-ssl-certificates-for-cross-cloud-scaling"></a>Настройка SSL-сертификатов для масштабирования в нескольких облаках

Очень важно обеспечить безопасность конфиденциальных данных, собранных веб-приложением, как при передаче в Базу данных SQL, так и при хранении в ней.

Вы настроите веб-приложения Azure и Azure Stack Hub для использования SSL-сертификатов для всего входящего трафика.

### <a name="add-ssl-to-azure-and-azure-stack-hub"></a>Добавление SSL в Azure и Azure Stack Hub

Добавление SSL в Azure:

1. Убедитесь, что получаемый SSL-сертификат является допустимым для созданного дочернего домена. (Можно воспользоваться групповыми сертификатами.)

2. На портале Azure следуйте инструкциям из разделов о **подготовке веб-приложения** и **привязке SSL-сертификата** статьи [Руководство по передаче и привязыванию SSL-сертификата в Службе приложений Azure](/azure/app-service/app-service-web-tutorial-custom-ssl). Выберите значение **SNI-based SSL** (SSL на основе SNI) в качестве **типа SSL**.

3. Перенаправьте весь трафик в HTTPS-порт. Следуйте инструкциям в разделе **Принудительное использование HTTPS** статьи [Руководство. Привязывание существующего настраиваемого SSL-сертификата к веб-приложениям Azure](/azure/app-service/app-service-web-tutorial-custom-ssl).

Чтобы добавить SSL в Azure Stack Hub, сделайте следующее:

1. Повторите шаги 1–3, которые использовались для Azure, с помощью портала Azure Stack Hub.

## <a name="configure-and-deploy-the-web-app"></a>Настройка и развертывание веб-приложения

Вы измените код приложения, чтобы обеспечить передачу данных телеметрии в правильный экземпляр Application Insights, и настроите веб-приложения с использованием правильных строк подключения. Дополнительные сведения о службе Application Insights см. в статье [Что такое Azure Application Insights?](/azure/application-insights/app-insights-overview)

### <a name="add-application-insights"></a>Добавление Application Insights

1. Откройте веб-приложение в Microsoft Visual Studio.

2. [Добавьте Application Insights](/azure/azure-monitor/app/asp-net-core#enable-client-side-telemetry-for-web-applications) в проект, чтобы передать данные телеметрии, с помощью которых Application Insights создает оповещения при увеличении или уменьшении веб-трафика.

### <a name="configure-dynamic-connection-strings"></a>Настройка динамических строк подключения

Каждый экземпляр веб-приложения будет использовать для подключения к Базе данных SQL разные методы. Приложение в Azure использует частный IP-адрес виртуальной машины SQL Server, а приложение в Azure Stack Hub — общедоступный IP-адрес виртуальной машины SQL Server.

> [!Note]  
> В интегрированной системе Azure Stack Hub общедоступный IP-адрес не должен маршрутизироваться через Интернет. В ASDK общедоступный IP-адрес не поддерживает маршрутизацию за пределы ASDK.

С помощью переменных Среды службы приложений вы можете передать другую строку подключения каждому экземпляру приложения.

1. Откройте приложение в Visual Studio.

2. Откройте файл Startup.cs и найдите такой блок кода:

    ```C#
    services.AddDbContext<MyDatabaseContext>(options =>
        options.UseSqlite("Data Source=localdatabase.db"));
    ```

3. Замените предыдущий блок кода следующим кодом, в котором используется строка подключения, определенная в файле *appsettings.json*:

    ```C#
    services.AddDbContext<MyDatabaseContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("MyDbConnection")));
     // Automatically perform database migration
     services.BuildServiceProvider().GetService<MyDatabaseContext>().Database.Migrate();
    ```

### <a name="configure-app-service-app-settings"></a>Настройка параметров Службы приложений

1. Создайте строки подключения для Azure и Azure Stack Hub. Эти строки должны быть одинаковыми за исключением используемых IP-адресов.

2. В Azure и Azure Stack Hub добавьте соответствующую строку подключения [в качестве параметра приложения](/azure/app-service/web-sites-configure) в веб-приложении, используя `SQLCONNSTR\_` в качестве префикса имени.

3. **Сохраните** параметры веб-приложения и перезагрузите приложение.

## <a name="enable-automatic-scaling-in-global-azure"></a>Включение автоматического масштабирования в глобальной среде Azure

При создании веб-приложения в Среде службы приложений запускается один его экземпляр. Его масштаб можно автоматически горизонтально увеличить, добавляя экземпляры, чтобы предоставить приложению дополнительные вычислительные ресурсы. Аналогичным образом можно автоматически горизонтально уменьшить масштаб и уменьшить число экземпляров, необходимых для приложения.

> [!Note]  
> Чтобы настроить горизонтальное увеличение и уменьшение масштаба, необходим план службы приложений. Если у вас нет плана, создайте его, прежде чем выполнять дальнейшие действия.

### <a name="enable-automatic-scale-out"></a>Включение автоматического развертывания

1. На портале Azure найдите план службы приложений для сайтов, которые нужно горизонтально масштабировать, а затем выберите **Увеличить масштаб (план службы приложений)** .

    ![Масштабирование службы приложений Azure](media/solution-deployment-guide-hybrid/image16.png)

2. Выберите **Включить автомасштабирование**.

    ![Включение автомасштабирования в Службе приложений Azure](media/solution-deployment-guide-hybrid/image17.png)

3. Введите имя для параметра **Имя параметра автомасштабирования**. Для правила автомасштабирования **по умолчанию** выберите параметр **Масштабировать на основе метрики**. Задайте для параметра **Ограничения экземпляров** значения **Минимум: 1**, **Максимум: 10** и **По умолчанию: 1**.

    ![Конфигурация автомасштабирования в Службе приложений Azure](media/solution-deployment-guide-hybrid/image18.png)

4. Выберите **+Add a rule** (+Добавление правила).

5. В разделе **Источник метрики** выберите **Current Resource** (Текущий ресурс). Используйте следующие критерии и действия для правила.

#### <a name="criteria"></a>Критерии

1. В разделе **Агрегат времени** выберите **Среднее**.

2. В разделе **Имя метрики** выберите **Процент ЦП**.

3. В разделе **Оператор** выберите **больше**.

   - Для параметра **Пороговое значение** задайте значение **50**.
   - Для параметра **Длительность** задайте значение **10**.

#### <a name="action"></a>Действие

1. В разделе **Операция** выберите **Увеличить счетчик на**.

2. Для параметра **Число экземпляров** задайте значение **2**.

3. Задайте для параметра **Восстановление** значение **5**.

4. Выберите **Добавить**.

5. Выберите **+ Add a rule** (+ Добавление правила).

6. В разделе **Источник метрики** выберите **Current Resource** (Текущий ресурс).

   > [!Note]  
   > Текущий ресурс будет содержать имя или GUID плана службы приложений, а раскрывающиеся списки **Тип ресурса** и **Ресурс** станут недоступными.

### <a name="enable-automatic-scale-in"></a>Включение автоматического горизонтального уменьшения масштаба

После уменьшения трафика веб-приложение Azure может автоматически сократить число активных экземпляров для снижения затрат. Это действие требует меньше ресурсов, чем развертывание, благодаря чему влияние на пользователей приложения сводится к минимуму.

1. Перейдите к условию горизонтального увеличения масштаба **По умолчанию**, а затем выберите **+ Add a rule** (Добавить правило). Используйте следующие критерии и действия для правила.

#### <a name="criteria"></a>Критерии

1. В разделе **Агрегат времени** выберите **Среднее**.

2. В разделе **Имя метрики** выберите **Процент ЦП**.

3. В разделе **Оператор** выберите **меньше**.

   - Для параметра **Пороговое значение** задайте значение **30**.
   - Для параметра **Длительность** задайте значение **10**.

#### <a name="action"></a>Действие

1. В разделе **Операция** выберите **Уменьшить счетчик на**.

   - Задайте для параметра **Число экземпляров** значение **1**.
   - Задайте для параметра **Восстановление** значение **5**.

2. Выберите **Добавить**.

## <a name="create-a-traffic-manager-profile-and-configure-cross-cloud-scaling"></a>Создание профиля диспетчера трафика и настройка его для масштабирования в нескольких облаках

Создайте на портале Azure профиль Диспетчера трафика, а затем настройте конечные точки, чтобы включить масштабирование в нескольких облаках.

### <a name="create-traffic-manager-profile"></a>Создание профиля диспетчера трафика

1. Выберите **Создать ресурс**.
2. Выберите **Сети**.
3. Выберите **Профиль диспетчера трафика** и настройте следующие параметры:

   - В поле **Имя** введите имя профиля. Это имя **должно** быть уникальным в пределах зоны trafficmanager.net. Оно используется для создания DNS-имени (например, northwindstore.trafficmanager.net).
   - Для параметра **Метод маршрутизации** выберите **Взвешенный**.
   - Из списка **Подписка** выберите подписку, в которой необходимо создать этот профиль.
   - В разделе **Группа ресурсов** создайте группу ресурсов, в которую следует поместить этот профиль.
   - Из списка **Расположение группы ресурсов** выберите расположение группы ресурсов. Этот параметр задает расположение группы ресурсов и не влияет на профиль диспетчера трафика, который развернут глобально.

4. Нажмите кнопку **создания**.

    ![Создание профиля диспетчера трафика](media/solution-deployment-guide-hybrid/image19.png)

   Когда глобальное развертывание профиля диспетчера трафика завершится, он появится в списке ресурсов для группы ресурсов, в которой он создан.

### <a name="add-traffic-manager-endpoints"></a>Добавление конечных точек диспетчера трафика

1. Найдите профиль диспетчера трафика, который вы создали. (Если вы перешли в группу ресурсов для профиля, выберите профиль.)

2. В колонке **Профиль диспетчера трафика** в разделе **Параметры** щелкните **Конечные точки**.

3. Выберите **Добавить**.

4. В разделе **Добавить конечную точку** используйте следующие параметры для Azure Stack Hub:

   - В поле **Тип** выберите **Внешняя конечная точка**.
   - Укажите **имя** конечной точки.
   - Для параметра **Полное доменное имя или IP-адрес** укажите внешний URL-адрес веб-приложения Azure Stack Hub.
   - Для параметра **Вес** оставьте значение по умолчанию **1**. В результате весь трафик будет поступать в эту конечную точку, если она работоспособна.
   - Оставьте флажок **Добавить как отключенный** снятым.

5. Нажмите кнопку **ОК**, чтобы сохранить конечную точку Azure Stack Hub.

Далее нужно настроить конечную точку Azure.

1. В разделе **Профиль диспетчера трафика** выберите **Конечные точки**.
2. Щелкните **+Добавить**.
3. В разделе **Добавить конечную точку** используйте следующие параметры для Azure:

   - В раскрывающемся списке **Тип** выберите **Конечная точка Azure**.
   - Укажите **имя** конечной точки.
   - В раскрывающемся списке **Тип целевого ресурса** выберите **Служба приложений**.
   - В разделе **Целевой ресурс** щелкните **Выберите службу приложений**, чтобы отобразился список веб-приложений, размещенных в этой подписке.
   - В колонке **Ресурсы** выберите службу приложений, которую требуется добавить в качестве первой конечной точки.
   - Для параметра **Вес** выберите значение **2**. В результате весь трафик будет передаваться в эту конечную точку, если основная конечная точка неработоспособна или если у вас настроено правило либо оповещение, которое при активации перенаправляет трафик.
   - Оставьте флажок **Добавить как отключенный** снятым.

4. Нажмите кнопку **ОК**, чтобы сохранить конечную точку Azure.

После настройки обе конечные точки отобразятся в разделе **Профиль диспетчера трафика**, если выбрать **Конечные точки**. В примере на следующем снимке экрана показано две конечные точки со сведениями о состоянии и конфигурации каждой из них.

![Конечные точки в профиле Диспетчера трафика](media/solution-deployment-guide-hybrid/image20.png)

## <a name="set-up-application-insights-monitoring-and-alerting-in-azure"></a>Настройка мониторинга и оповещений Application Insights в Azure

Azure Application Insights позволяет отслеживать приложение и отправлять оповещения на основе настраиваемых условий (например, при недоступности приложения, возникновении ошибок или проблемах с производительностью).

Для создания оповещений вы будете использовать метрики Azure Application Insights. Когда эти оповещения активируются, экземпляр веб-приложений автоматически переключится с Azure Stack Hub на Azure для горизонтального увеличения масштаба, а затем обратно на Azure Stack Hub для горизонтального уменьшения масштаба.

### <a name="create-an-alert-from-metrics"></a>Создание оповещения из метрик

В рамках этого руководства перейдите к группе ресурсов на портале Azure, а затем выберите экземпляр Application Insights, чтобы открыть **Application Insights**.

![Application Insights](media/solution-deployment-guide-hybrid/image21.png)

С помощью этого представления вы создадите оповещения о горизонтальном увеличении и уменьшении масштаба.

### <a name="create-the-scale-out-alert"></a>Создание оповещения о развертывании

1. В меню **Настройка** выберите **Оповещения (классические)** .
2. Выберите **Добавить оповещение метрики (классическое)** .
3. В разделе **Добавление правила** настройте следующие параметры:

   - Для параметра **Имя** введите **Burst into Azure Cloud** (Повышение в облако Azure).
   - Поле **Описание** заполнять необязательно.
   - В разделе **Источник** > **Оповещения включены** выберите **Метрики**.
   - В разделе **Критерии** выберите свою подписку, группу ресурсов для профиля диспетчера трафика, а также имя профиля диспетчера трафика для ресурса.

4. Для параметра **Метрика** выберите **Частота запросов**.
5. Для параметра **Условие** выберите **больше**.
6. Для параметра **Пороговое значение** введите **2**.
7. Для параметра **Период** выберите **За последние 5 минут**.
8. В разделе **Уведомить по**:
   - Установите флажок **Участники, читатели и владельцы электронной почты**.
   - Введите адрес электронной почты в разделе **Дополнительные адреса электронной почты администратора**.

9. В строке меню выберите **Сохранить**.

### <a name="create-the-scale-in-alert"></a>Создание оповещения о горизонтальном уменьшении масштаба

1. В меню **Настройка** выберите **Оповещения (классические)** .
2. Выберите **Добавить оповещение метрики (классическое)** .
3. В разделе **Добавление правила** настройте следующие параметры:

   - В поле **Имя** введите **Scale back into Azure Stack Hub** (Свернуть обратно в Azure Stack Hub).
   - Поле **Описание** заполнять необязательно.
   - В разделе **Источник** > **Оповещения включены** выберите **Метрики**.
   - В разделе **Критерии** выберите свою подписку, группу ресурсов для профиля диспетчера трафика, а также имя профиля диспетчера трафика для ресурса.

4. Для параметра **Метрика** выберите **Частота запросов**.
5. Для параметра **Условие** выберите **меньше**.
6. Для параметра **Пороговое значение** введите **2**.
7. Для параметра **Период** выберите **За последние 5 минут**.
8. В разделе **Уведомить по**:
   - Установите флажок **Участники, читатели и владельцы электронной почты**.
   - Введите адрес электронной почты в разделе **Дополнительные адреса электронной почты администратора**.

9. В строке меню выберите **Сохранить**.

На следующем снимке экрана показаны оповещения для развертывания и свертывания.

   ![Оповещения Application Insights (классическое приложение)](media/solution-deployment-guide-hybrid/image22.png)

## <a name="redirect-traffic-between-azure-and-azure-stack-hub"></a>Перенаправление трафика между Azure и Azure Stack Hub

Вы можете настроить для трафика веб-приложения автоматическое переключение или переключение вручную между Azure и Azure Stack Hub.

### <a name="configure-manual-switching-between-azure-and-azure-stack-hub"></a>Настройка переключения вручную между Azure и Azure Stack Hub

По достижении на веб-сайте пороговых значений, которые вы настроили, вы получите оповещение. Для перенаправления трафика в Azure вручную сделайте следующее.

1. На портале Azure выберите профиль диспетчера трафика.

    ![Конечные точки Диспетчера трафика на портале Azure](media/solution-deployment-guide-hybrid/image20.png)

2. Выберите **Конечные точки**.
3. Выберите **Конечная точка Azure**.
4. В разделе **Состояние** выберите **Включено** и нажмите **Сохранить**.

    ![Включение конечной точки Azure на портале Azure](media/solution-deployment-guide-hybrid/image23.png)

5. На странице **Конечные точки** профиля диспетчера трафика выберите **Внешняя конечная точка**.
6. В разделе **Состояние** выберите **Отключено** и нажмите **Сохранить**.

    ![Отключение конечной точки Azure Stack Hub на портале Azure](media/solution-deployment-guide-hybrid/image24.png)

После настройки конечных точек трафик приложения передается в веб-приложение развертывания Azure, а не в веб-приложение Azure Stack Hub.

 ![Изменение конечных точек в трафике веб-приложения Azure](media/solution-deployment-guide-hybrid/image25.png)

Для возвращения потока обратно в Azure Stack Hub используйте предыдущие шаги, чтобы:

- включить конечную точку Azure Stack Hub;
- отключить конечную точку Azure.

### <a name="configure-automatic-switching-between-azure-and-azure-stack-hub"></a>Настройка автоматического переключения между Azure и Azure Stack Hub

Вы также можете использовать мониторинг Application Insights, если приложение выполняется в [бессерверной](https://azure.microsoft.com/overview/serverless-computing/) среде, предоставляемой Функциями Azure.

В этом случае можно настроить Application Insights, чтобы использовать веб-перехватчик, который вызывает приложение-функцию. Это приложение автоматически включает или отключает конечную точку в ответ на оповещение.

Следуйте приведенным ниже инструкциям в качестве руководства для настройки автоматического переключения трафика.

1. Создайте приложение-функцию Azure.
2. Создайте функцию, активируемую HTTP-запросом.
3. Импортируйте пакеты SDK для Resource Manager, веб-приложений и диспетчера трафика.
4. Разработайте код, чтобы:

   - пройти проверку подлинности в подписке Azure;
   - использовать параметр, который переключает конечные точки диспетчера трафика, чтобы направить трафик в Azure или Azure Stack Hub.

5. Сохраните код и добавьте URL-адрес приложения-функции с соответствующими параметрами в раздел **Веб-перехватчик** с параметрами правила генерации оповещений Application Insights.
6. Трафик будет автоматически перенаправлен, когда сработает оповещение Application Insights.

## <a name="next-steps"></a>Дальнейшие действия

- Дополнительные сведения о шаблонах для облака Azure см. в статье [Конструктивные шаблоны облачных решений](/azure/architecture/patterns).
