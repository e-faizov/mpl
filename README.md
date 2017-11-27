# mpl (maratl plugin library)

требования: не ниже Visual Studio 2013

для использования и написания плагина требуется:
 - создать обычный с++ проект для dll
 - (опционально) в настройках проекта для release и debug поставить настройки /MT и /MTd соответственно (configuration properties -> C/C++ -> Code Generation -> Runtime Library)
 - создать класс, который будет являться плагином (например CPlugin) 
 - подключить plugin.h
 - отнаследоваться от mpl::plugin и обязательно написать реалтзацию методов Init, getName, getVersion и опционально методов InitData, getDescription, Uninit
 - в cpp файле добавить PLUGIN_DECL с указанием класса плагина и идентификатором класса (пример PLUGIN_DECL(CPlugin, "{b67a2317-e548-47e5-946e-a7c1c7c0191a}")), clsid должен быть уникальным и быть заново сгенерированным
 - написать плагин используя api!
 
для запуска:
 - специальный маратл, позволяющий изменять plc.data
 - создать папку plugins рядом с maratl.exe
 - положить dll файл плагина в папку plugins
 - в файл plc.data (лежит в папке config) добавить информацию о плагине (пример файла в example_plc_data.txt)
 - в тэге file аттрибут path указывать на расположение файла относительно папки plugins
 - в тэге plugin аттрибут clsid, берется из PLUGIN_DECL
 - запустить maratl
 
##api
####mpl::logger
 инструмент для логирования, позволяет писать из плагина в лог maratl
 
```c++
bool CPlugin::Init(std::shared_ptr<mpl::maratl> maratl)
{
	auto log = maratl->getLogger();
	log->log(L"log string");
}
```

####mpl::uiCommands
 - ниструмент для добавления команд интерфейсу
 - метод create позволяет зарегистрировать обработчик и создает функцию ответа
 - maratl сам обрезает и подставляет namespace, при работе с командами в плагине его стоит опустить
 
```c++
//c++
bool CPlugin::Init(std::shared_ptr<mpl::maratl> maratl)
{
	auto cmdServ = maratl->getUiCommands();
	m_recvFn = cmdServ->create(L"plgNamespace", [=](const std::wstring& name, const std::wstring& val) -> bool
	{
		if (name == L"testCmd")
		{
			if (m_recvFn)
				m_recvFn(L"testRecv", L"done");
			log->log(val);
		}
		return true;
	});
}
```

```js
//js
function OnResponse(name, data)
{
	if(name === "plgNamespace.testRecv")
	{
		//получени ответа от плагина
	}
}
terminal.OnResponse = OnResponse;
terminal.ProcessCommand("plgNamespace.testCmd", "data");

```
####mpl::network
 - инструмент позволяет отправлять запросы, как на сервисы qiwi с авторизацией, так и на другие сервисы без авторизации
 - метода create создает объект запроса и принимает флаг, создать запрос с авторизацеией или без
 - в случае если запрос с авторизацей, в качестве url можно указать только сервисы qiwi и обязательно ssl(https://*.qiwi.com)

```c++
//c++
bool CPlugin::Init(std::shared_ptr<mpl::maratl> maratl)
{
	auto net = maratl->getNetwork();
	auto req = net->create(false);

	req->setMethod(L"POST");
	req->setUrl(L"http://google.com");
	req->send(L"request body", [=](const std::wstring& response, int httpStatus, int transportError, const std::wstring& tag)
	{
		log->log(response);
	});
}
```
 
####mpl::timers
 - позволяет зарегистрировать таймер, который будет срабатывать раз в 30 секунд и каждый в своем потоке
 
```c++
//c++
bool CPlugin::Init(std::shared_ptr<mpl::maratl> maratl)
{
	auto timers = maratl->getTimers();
	timers->addTimer([]()
	{
		//обработка таймера
	}
}
```

####mpl::devices
 - дает возможность добавть устройство в поиск maratl при старте и вывести в лог и синие окно, если устройство найдено
 - полсе создания устройства следует добавить нужную функцию для поиска, через RS(onDetectRS) или через другие каналы(onDetect), по умолчанию, если функция не добавлена, возвращается false
 
```c++
//c++
bool CPlugin::Init(std::shared_ptr<mpl::maratl> maratl)
{
	auto devices = maratl->getDevices();
	auto dev = devices->create(L"Device name");
	
	dev->onDetectRS([=](int iComPortNumber) -> bool {
		//поиск устойства на com порте, с передачей номера порта
		log->log(L"поиск устройства");
		return false;
	});

	dev->onDetect([]() -> bool {
		//поиск устройства, если оно не на com порте (usb, net, shared file, etc)
		return true;
	});
}
```
 
 
 
 
 
 
