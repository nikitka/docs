# Yandex Cloud Interconnect

{{ interconnect-full-name }} позволяет установить приватное выделенное соединение между вашей локальной сетевой инфраструктурой и {{ yandex-cloud }}. Выделенное соединение надежнее, быстрее и безопаснее подключения через интернет.

## Магистральный канал {#trunk-link}

Основой решения {{ interconnect-name }} служит магистральный канал (транк-соединение) с пропускной способностью от 100 Мбит/с до 10 Гбит/с и более. Для организации магистрального канала используется оптический стык стандарта 10Ge LR-LC с оборудованием {{ yandex-cloud }}. Поверх магистрального канала создаются выделенные соединения.

Точки присутствия {{ yandex-cloud }}:
* ММТС-9, Москва, ул. Бутлерова, д. 7.
* StoreData, Москва, ул. Нижегородская, д. 32, стр. А.
* Dataline NORD, Москва, Коровинское шоссе, д. 41.

## Приватное соединение {#private-connection}

Приватное соединение настраивается поверх магистрального канала и обеспечивает связность между вашей локальной сетевой инфраструктурой и виртуальной сетью в {{ yandex-cloud }}. В целях резервирования к одной виртуальной сети можно подключить несколько выделенных соединений.

Изоляция двух выделенных соединений в одном магистральном канале происходит посредством тегирования трафика с помощью VLAN.

Для обмена маршрутной информацией и дальнейшей передачи трафика необходимо настроить протокол [BGP](https://ru.wikipedia.org/wiki/Border_Gateway_Protocol). Количество принимаемых префиксов ограничено: можно анонсировать не более 500 маршрутов. В случае превышения порога BGP-сессия будет сброшена и инициализирована заново.

При организации приватного соединения рекомендуется произвести резервирование магистрального канала через несколько точек присутствия. Резервирование доступа к VPC через одну площадку не поддерживается, как и протоколы отказоустойчивости маршрутизаторов (например CARP, HSRP, VRRP), так как эти методы не повышают надежность по сравнению с одним приватным каналом.

В рамках одной площадки допускается использовать:
* методы агрегации линков (LACP), в том числе в режиме active/passive;
* стекирование коммутаторов на стороне пользователя при условии, что коммутаторы будут объединены в единое логическое устройство.

Поверх одного магистрального канала можно создать [два](../concepts/limits#yandex-cloud-interconnect) приватных соединения.

### Организация приватного соединения {#set-up-private-connection}

Чтобы воспользоваться приватным соединением, необходимо подать заявку. В заявке должна быть выбрана точка присутствия, где будет настраиваться соединение, и его требуемая пропускная способность. Если вы организуете канал через партнера, укажите название организации-партнера. 

После обработки заявки с вами свяжутся наши специалисты для обсуждения деталей. Услуга предоставляется при наличии технической возможности. 

Чтобы соединение заработало, нужно настроить связность вашей локальной сетевой инфраструктуры и виртуальной сети в {{ yandex-cloud }}:

1. [Создайте](../quickstart.md) в облаке виртуальную сети и подсети, к которым будет подключена локальная сетевая инфраструктура.
1. Согласуйте с {{ yandex-cloud }} следующие параметры:
   * С вашей стороны: 
     * Идентификатор облака.
     * Идентификатор виртуальной сети.
     * CIDR пиринговой подсети из диапазона приватных адресов [RFC1918](https://tools.ietf.org/html/rfc1918).
     * IP-адреса BGP-пиров на стороне пользователя и на стороне {{ yandex-cloud }}.
     * BGP ASN.
   * Со стороны {{ yandex-cloud }}:
     * Идентификатор VLAN.