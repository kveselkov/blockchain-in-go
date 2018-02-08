Часть 2: Сетевое взаимодействие — Написание blockchain менее чем за 200 строк кода на Go
===
<img src="https://habrastorage.org/webt/6n/bz/q9/6nbzq9ptgropjb-ejnwlbur9q9g.png" alt="image"/>
Вы прочитали [первую часть](https://habrahabr.ru/post/347930/) из этой серии? Если нет, то стоит взглянуть. Не волнуйся, мы подождем...
<cut/>
Добро пожаловать!

Мы были ошеломлены обратной связью от нашего первого поста: "Написание blockchain менее чем за 200 строк кода на Go". То, что предназначалось для небольшого урока для начинающих разработчиков по blockchain, приобрело новую жизнь. Нас завалили запросами сделать пост, где мы добавляем сетевое взаимодействие.

Прежде чем начнем, вы можете присоединиться к нашему чату в [Telegram](https://t.me/joinchat/FX6A7UThIZ1WOUNirDS_Ew)! Это лучшее место, что бы задать нам вопросы, дать отзывы и попросить новые уроки. Если вы нуждаетесь в помощи с вашим кодом, то это идеальное место, что бы спросить!

Последний пост показал вам, как создать свой собственный blockchain с хэшированием и валидацией каждого нового блока. Но все это выполнялось на одной ноде. Как мы можем подключить еще одну ноду к нашему основному приложению и создавать новые блоки, и обновлять всю цепочку блоков на всех остальных нодах?

Рабочий процесс
---
<img src="https://habrastorage.org/webt/8j/bc/qa/8jbcqass8xws_aspo-ugpiyehik.png" alt="image"/>
* Первый терминал создает первый базовый блок и TCP сервер, к которому могут подключаться новые ноды.

*Шаг 1*

* Открываются дополнительные терминалы и TCP соединения с первым терминалом
* Новый терминал записывает блок на первый терминал

*Шаг 2*

* Первый терминал проверяет блок
* Первый терминал рассылает новую цепочку блоков каждой новой ноде

*Шаг 3*

* Все терминалы синхронизированы!

После урока попробуйте сделать сами: каждый новый терминал, так же выступает в качестве "первых" терминалов, но с различными TCP портами и каждый с каждым имеет соединения для правильной работы сети.

Что вы сможете сделать
---
* Запустите терминал, который обеспечивает первый базовый блок
* Запустите множество новых дополнительных терминалов, сколько хотите и пусть они запишут блоки в первый терминал
* Первый терминал должен рассылать обновленные блоки для новых терминалов

Что вы не сможете сделать
---
Как и в предыдущем посте, цель данного урока в том, что бы получить базовую сеть из нод, что бы вы дальше смогли самостоятельно изучать blockchain. Вы не сможете добавить компьютеры из другой сети, которые будут писать в ваш первый терминал, но этого можно достичь, запустив бинарник в облаке. Кроме того, цепочка блоков будет смоделирована для каждой из нод. Не волнуйтесь, мы скоро все объясним.

Давайте начнем кодить!
---
Местами будет обзор предыдущего поста. Оставим множество функций, таких как генерация блоков, хэширование, проверка. Функционал HTTP использовать не будем, а просматривать результат будем в консоли, а для работы по сети будем использовать TCP.

Какие различия между TCP и HTTP?

Не будем вдаваться в подробности, но все, что вам нужно знать, что TCP является базовым протоколом, который передает данные. HTTP построен поверх TCP стека, что бы использовать эту передачу данных между интернетом и браузером. Когда вы просматриваете веб-сайт, вы используете HTTP протокол. В данном уроке будем работать с TCP, так как нам не нужно ничего просматривать в браузере. Go имеет хороший [сетевой](https://golang.org/pkg/net/) пакет, предоставляющий все функции TCP соединения, которые нам необходимы.

Установка, импорты и обзор
---
Некоторая  реализация уже рассматривались в [первой](https://habrahabr.ru/post/347930/) части. Для генерации и проверки цепочки блоков будем использовать функции из предыдущей статьи.

*Установка*

Создайте ```.env``` файл в вашей главной директории и добавьте строку:
```
ADDR=9000
```
Сохраняем номер порта TCP, который хотим использовать (в нашем случае 9000) в переменной окружения под названием ```ADDR```.

Если вы еще этого не сделали, установите следующие пакеты:
```
go get github.com/davecgh/go-spew/spew
```
для красивой печати нашей цепочки блоков в консоль
```
go get github.com/joho/godotenv
```
для чтения переменных из нашего ```.env``` файла.
Создайте пустой ```main.go``` файл. Там расположим наш код.

*Импорты*

Декларация пакета и необходимые нам импорты:
```
package main

import (
	"bufio"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"log"
	"net"
	"os"
	"strconv"
	"time"

	"github.com/davecgh/go-spew/spew"
	"github.com/joho/godotenv"
)
```

*Обзор*
Следующие фрагменты кода хорошо описаны в [первой](https://habrahabr.ru/post/347930/) части.

Давайте создадим нашу ```Block``` структуру и объявим слайс блоков, это и будет наша цепочка блоков.
```
type Block struct {
	Index     int
	Timestamp string
	BPM       int
	Hash      string
	PrevHash  string
}

var Blockchain []Block
```
Объявим так же нашу функцию хэширования, которая понадобится нам при создании новых блоков.
```
func calculateHash(block Block) string {
	record := string(block.Index) + block.Timestamp + string(block.BPM) + block.PrevHash
	h := sha256.New()
	h.Write([]byte(record))
	hashed := h.Sum(nil)
	return hex.EncodeToString(hashed)
}
```
И функция создания блоков:
```
func generateBlock(oldBlock Block, BPM int) (Block, error) {

	var newBlock Block

	t := time.Now()

	newBlock.Index = oldBlock.Index + 1
	newBlock.Timestamp = t.String()
	newBlock.BPM = BPM
	newBlock.PrevHash = oldBlock.Hash
	newBlock.Hash = calculateHash(newBlock)

	return newBlock, nil
}
```
Можем убедиться, что наш новый блок правильный, для этого проверим, что поле ```PrevHash``` ссылается на поле ```Hash``` из предыдущего блока.
```
func isBlockValid(newBlock, oldBlock Block) bool {
	if oldBlock.Index+1 != newBlock.Index {
		return false
	}

	if oldBlock.Hash != newBlock.PrevHash {
		return false
	}

	if calculateHash(newBlock) != newBlock.Hash {
		return false
	}

	return true
}
```
Теперь гарантируем, что возьмем самую длинную цепочку, как правильную:
```
func replaceChain(newBlocks []Block) {
	if len(newBlocks) > len(Blockchain) {
		Blockchain = newBlocks
	}
}
```
Замечательно! Получили основной функционал по работе с цепочкой блоков. Теперь можем перейти к созданию взаимодействия по сети.

Сетевое взаимодействие
---
Давайте создадим сеть, которая может передавать новые блоки, интегрировать их в цепочку и транслировать новую цепочку для сети.

Начнем с функции main, но перед этим давайте объявим глобальную переменную ```bcServer```, которая является каналом принимающим входящие блоки.
```
var bcServer chan []Block
```
*Примечание: Каналы являются одним из наиболее популярных инструментов в Go и обеспечивает красивую реализацию чтения/записи данных и чаще всего используется для предотвращения состояния гонки. Они становятся мощным инструментом, когда несколько Go-рутин конкурентно (не путать с параллельностью) пишут в один и тот же канал. Обычно в Java и других C-подобных языках вам придется блокировать и разблокировать мьютекс для доступа к данным. Каналы в Go делают это намного проще, хотя в Go так же есть и мьютексы. Можете подробнее почитать об этом [тут](https://golangbot.com/channels/).*

Теперь давайте объявим нашу функцию main и загрузим переменные окружения из нашего файла ```.env```, который находится в корневом каталоге. А так же запустим экземпляр нашего ```bcServer``` в функции main.
```
func main() {
   err := godotenv.Load()
   if err != nil {
	   log.Fatal(err)
   }

   bcServer = make(chan []Block)

   // create genesis block
   t := time.Now()
   genesisBlock := Block{0, t.String(), 0, "", ""}
   spew.Dump(genesisBlock)
   Blockchain = append(Blockchain, genesisBlock)
}
```
Теперь нам необходимо создать TCP сервер. Помните, что TCP серверы похожи на HTTP серверы, но для работы с браузером протокола TCP недостаточно. Все данные будут отображаться через консоль. Будем обрабатывать несколько соединений с нашим TCP портом. Добавим это к нашей функции main:
```
server, err := net.Listen("tcp", ":"+os.Getenv("ADDR"))
	if err != nil {
		log.Fatal(err)
	}
	defer server.Close()
```
Этот код запустит наш TCP сервер на порту 9000. Важно выполнить ```defer server.Close()```, что бы соединение закрылось, когда больше нет необходимости в нем. Почитать подробнее про ```defer``` можно [тут](https://blog.golang.org/defer-panic-and-recover).

Теперь нам необходимо создавать новое соединение каждый раз, когда получаем запрос на установку соединения, и нам необходимо будем его обработать. Добавим еще кода:
```
for {
	conn, err := server.Accept()
	if err != nil {
		log.Fatal(err)
	}
	go handleConn(conn)
}
```
Создаем бесконечный цикл, в котором принимаем новые соединения. Для конкурентной обработки, каждое соединение запускаем в  обработчике в Go рутине ```go handleConn(conn)```, поэтому не останавливаем наш цикл. Таким образом можем одновременно слушать несколько соединений конкурентно.

Внимательный читатель заметит, что функция обработчик ```handleConn``` не объявлена. Мы пока что создали нашу основную функцию ```main```. Целиком она выглядит так:
```
func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal(err)
	}

	bcServer = make(chan []Block)

	// create genesis block
	t := time.Now()
	genesisBlock := Block{0, t.String(), 0, "", ""}
	spew.Dump(genesisBlock)
	Blockchain = append(Blockchain, genesisBlock)

	// start TCP and serve TCP server
	server, err := net.Listen("tcp", ":"+os.Getenv("ADDR"))
	if err != nil {
		log.Fatal(err)
	}
	defer server.Close()

	for {
		conn, err := server.Accept()
		if err != nil {
			log.Fatal(err)
		}
		go handleConn(conn)
	}

}
```
Давайте теперь напишем нашу функцию ```handleConn```.  Она принимает только один аргумент, это интерфейс ```net.Conn```. На наш взгляд, интерфейсы в Go поразительны и они отличают его от всех C-подобный языков. Конкурентность и Go рутины рекламируют язык, но интерфейсы и тот факт, что они могут реализовывать интерфейс не явно, является самой мощной функцией языка. Если вы еще не используете интерфейсы в Go, ознакомитесь с ними как только сможете. Интерфейсы это ваш следующий шаг, для становления как Go разработчика.

Поместите в заготовку функции обработчика отложенное закрытие соединения ```defer```, что бы не забыть его закрыть, когда завершим работу.
```

func handleConn(conn net.Conn) {
	defer conn.Close()
}
```
Теперь нам нужно разрешить клиенту добавлять новые блоки в цепочку. Для данных будем использовать частоту пульса, как в [первой](https://habrahabr.ru/post/347930/) части. Замерьте свой пульс в течение минуты и запомните это число. Это будет параметр BPM (beats per minute)

Для реализации вышеуказанного нам необходимо:
* попросить клиента ввести свой BPM
* запросить это значение у клиента через ```stdin```
* создать новый блок с этими данными, используя функции generateBlock, isBlockValid и replaceChain
* положить новую цепочку блоков в канал, созданный ранее для передачи по сети
* разрешить клиенту вводить новое значение BMP

Код, который реализует выше описанный функционал:
```
io.WriteString(conn, "Enter a new BPM:")

scanner := bufio.NewScanner(conn)


go func() {
	for scanner.Scan() {
		bpm, err := strconv.Atoi(scanner.Text())
		if err != nil {
			log.Printf("%v not a number: %v", scanner.Text(), err)
			continue
		}
		newBlock, err := generateBlock(Blockchain[len(Blockchain)-1], bpm)
		if err != nil {
			log.Println(err)
			continue
		}
		if isBlockValid(newBlock, Blockchain[len(Blockchain)-1]) {
			newBlockchain := append(Blockchain, newBlock)
			replaceChain(newBlockchain)
		}

		bcServer <- Blockchain
		io.WriteString(conn, "\nEnter a new BPM:")
	}
}()
```
Создаем новый сканер. ```for scanner.Scan()``` это цикл, который работает конкурентно в Go рутине и отдельно от других соединений. Делаем быстрое строковое преобразование значения BMP (которое всегда будет типом ```integer```, поэтому проверяем его). Выполняем стандартную генерацию блока, проверка блока на валидность и добавление нового блока в цепочку.

Синтаксис ```bcServer <- Blockchain```  просто означает, что мы бросаем нашу новую цепочку в канал, который создали ранее. Затем предлагаем клиенту ввести новое значение BPM для создания следующего блока.

*Широковещательный канал*
Нам необходимо разослать новую цепочку блоков для всех соединений на нашем TCP сервере. Поскольку мы программируем на одном компьютере, нам надо имитировать, как данные будет передаваться всем клиентам. В функцию ```handleConn``` нам необходимо добавить:
* конвертацию нашей цепочки блоков в JSON формат, что она была удобочитаемой
* запись новой цепочки блоков в консоль для каждого из соединений
* установка таймаута периодичности, что бы наша цепочка блоков не спамила постоянно. В существующих системах это происходит каждый X минут. Будем использовать 30 секунд
* вывод основной цепочку блоков на первом терминале, что бы мы могли видеть, что происходит. Так мы убедимся, что блоки добавляемые различными нодами, действительно интегрируются в основную цепочку блоков

Вот код, выполняющий все в нужном порядке:
```
go func() {
	for {
		time.Sleep(30 * time.Second)
		output, err := json.Marshal(Blockchain)
		if err != nil {
			log.Fatal(err)
		}
		io.WriteString(conn, string(output))
	}
}()

for _ = range bcServer {
	spew.Dump(Blockchain)
}
```
Замечательно! Наша функция ```handleConn``` готова. Фактически вся программа готова и мы сохранили ее компактность в 200 строк кода. Это неплохо, не так ли?

Целиком со всем кодом, можно ознакомиться [тут](https://github.com/mycoralhealth/blockchain-tutorial/blob/master/networking/main.go)!

Интересный материал
---
Давайте перейдем в директорию с нашим кодом и запустим нашу программу, выполнив: ```go run main.go```
<img src="https://habrastorage.org/webt/vz/jl/yi/vzjlyiro3g-mgc4xybyuzladihc.png" alt="image"/>
Как и ожидалось, видим наш базовый блок. В это же время запустили TCP сервер на порту 9000, который может принимать несколько соединений.

Откройте новое окно терминала и подключитесь к нашему TCP серверу с помощью ```nc localhost 9000```. Будем использовать разный цвет в терминалах, что бы было понятно, что это разные клиенты. Сделайте это несколько раз с разными сеансами терминала, что бы запустить несколько клиентов.
<img src="https://habrastorage.org/webt/hx/m7/2f/hxm72fj5zigqqz6276__bocuof0.png" alt="image"/>
<img src="https://habrastorage.org/webt/vz/jl/yi/vzjlyiro3g-mgc4xybyuzladihc.png" alt="image"/>
<img src="https://habrastorage.org/webt/nt/6w/qn/nt6wqn6lprwpm_m6l6ljm257zio.png" alt="image"/>
Теперь введите BPM на любом из клиентов. Видим, что новый блок добавлен в первый терминал! Сеть работает!
<img src="https://habrastorage.org/webt/a_/sv/wg/a_svwgyuipnki7pv-_eryb-6of0.png" alt="image"/>
<img src="https://habrastorage.org/webt/wq/1q/y5/wq1qy56bauup-tkwl-djeqvrfbq.png" alt="image"/>
Ждем 30 секунд. Перейдите к одному из других клиентов, и вы увидите, что новая цепочка блоков передалась всем клиентам, даже если эти клиенты никогда не вводили BPM!
<img src="https://habrastorage.org/webt/kx/pz/qx/kxpzqxnecf4ff4n0n44brzzen7e.png" alt="image"/>

Следующие шаги
---
Поздравляем! Вы не только создали свой собственный blockchain из последнего урока, но и добавили сетевое взаимодействие. Теперь есть несколько направлений для того, что бы двигаться дальше:
* что бы получить большую сеть, работающую локально, создайте несколько каталогов, в которых хранится копия приложения. Каждая копия должна иметь разные TCP-порты. Для каждого сеанса терминала слушайте порт TCP и подключайтесь к другому, что бы вы могли получать и отправлять ваши данные.
* Объединяйте данные, полученные с нескольких портов. Это тема для другого урока, но это делается легко. Все это blockchain. Он должен принимать данные, а также передавать данные.
* Вы можете попробовать это с друзьями, настройте сервер в облаке, используя ваш любимый хостинг-провайдер. Попросите ваших друзей подключиться к нему и отправить данные. На данном этапе можно добавить немного безопасности. Если  будут запросы, то мы сделаем учебник по этому материалу.
Вы уже понимаете множество аспектов blockchain. Рекомендуем почитать алгоритмы наподобие Proof of Work или Proof of Stake.
Или можете просто подождать, пока мы напишем новое сообщение в блоге :-)
Напомним, что бы сообщить нам, что вы хотите увидеть, присоединяйтесь к нашему [Telegram](https://t.me/joinchat/FX6A7UThIZ1WOUNirDS_Ew) чату! Возможно, вы сможете поменять наше мнение о том, что написать в дальнейших уроках. Можете подписаться на наш [Twitter](https://twitter.com/myCoralHealth) так же.
Чтобы узнать больше о Coral Health и о том, как мы используем blockchain в исследовательской работе по медицине, можете посетить наш [сайт](https://mycoralhealth.com/).

*P.S. Автор перевода будет благодарен за указанные ошибки и неточности перевода.*