## AULA 1

Para instalar o GO no Windows foi utilizado este endereço:

https://go.dev/doc/install

Para criar a aplicação

```go mod init <<repositorio>>```

Exemplo:

```go mod init github.com/rocketseat-education/semana-tech-go-react-server```

Para persistir os dados da aplicação, foi utilizado um banco de dados Postgres. Para isso, foi utilizado Docker e containeres foram definidos num arquivo compose.yml, localizado na raiz da aplicação.

Depois de iniciar o Docker Desktop, foi executado o seguinte comando:

```docker compose up``` 

Caso seja necessário parar o serviço:

```docker compose down```


Para criar as migrations do banco de dados foi utilizado o Tern.

```go install tern```

A pasta onde isso vai ficar armazenado é a internal/pgstore. Depois de criar estas pastas, rodar o seguinte comando:

```turn init ./internal/store/pgstore/migrations```

O arquivo tern.conf é usado para configurar a migration.

A aplicação vai ter duas tabelas. Para criar as migrations para cada tabela, rodar os comandos:

```
turn new --migrations ./internal/store/pgstore/migrations create_rooms_table
turn new --migrations ./internal/store/pgstore/migrations create_messages_table
```

Os dados para acessar o banco de dados ficam num arquivo .env. Para o Tern tem acesso as credenciais do banco, será necessário criar um command.

na raiz do projeto,  criar a pasta cmd/tools/terndotenv e inserir dentro um arquivo chamado main.go

Para executar as migrações, subir o docker e depois rodar o comando a seguir:

```go run cmd/tools/terndotenv/main.go```

Para que o Go utilize o banco de dados Postgres será necessário instalar um pacote chamado sqlc.

```go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest```

Será necessário criar o arquivo ./internal/store/pgstore/sqlc.yml. Ele será criado com o seguinte comando:

```sqlc generate ./internal/store/pgstore/sqlc.yml```

Para atualizar as dependências:

```
go mod tidy
go generate
```

Dentro da pasta cmd, criar a pasta wsrs e dentro o arquivo main.go. Este será o arquivo que será utilizado para subir a aplicação.

## AULA 2

As rotas são criadas no arquivo internal/api/api.go 

Neste arquivo são ser instalados os pacotes middlewares, RequestID, Recoverer e Logger. Também será necessário instalar o CORS, para permitir o acesso desta API. Para instalar o CORS primeiro é necessário executar o seguinte comando:

```go get github.com/go-chi/cors```

Outro pacote que será necessário instalar é o WebSockets

```go get github.com/gorilla/websocket```

Para instalar dependências que estiverem faltando, utilizar o comando:

```go tidy```

Estrutura do arquivo api.go

```
package api

import (
 .... dependencias do módulo
)

type apiHandler struct {
	q           *pgstore.Queries
	r           *chi.Mux
	upgrader    websocket.Upgrader
	subscribers map[string]map[*websocket.Conn]context.CancelFunc
	mu          *sync.Mutex
}

func (h apiHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	h.r.ServeHTTP(w, r)
}

func NewHandler(q *pgstore.Queries) http.Handler {
	a := apiHandler{
		q:           q,
		upgrader:    websocket.Upgrader{CheckOrigin: func(r *http.Request) bool { return true }},
		subscribers: make(map[string]map[*websocket.Conn]context.CancelFunc),
		mu:          &sync.Mutex{},
	}

	r := chi.NewRouter()
	r.Use(middleware.RequestID, middleware.Recoverer, middleware.Logger)

	r.Use(cors.Handler(cors.Options{
		AllowedOrigins:   []string{"https://*", "http://*"},
		AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"},
		AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
		ExposedHeaders:   []string{"Link"},
		AllowCredentials: false,
		MaxAge:           300,
	}))
	
	(.... rotas da aplicação  ...)

	a.r = r
	return a
}
```

A criação de uma rota segue o formato a seguir, perceba quwe no código abaixo temos o método HTTP, o enderço da rota e a função que será executada quando a rota for acessada

```
r.Get("/subscribe/{room_id}", a.handleSubscribe)
```

É possível também agrupar rotas, como no exemplo a seguir:

```
r.Route("/api", func(r chi.Router) {
	r.Route("/rooms", func(r chi.Router) {
		r.Post("/", a.handleCreateRoom)
		r.Get("/", a.handleGetRooms)
		(.. outras rotas...)
	}
}		
```

Um exemplo de função handler usada para receber ou retornar dados de uma determinada rota.

```
func (h apiHandler) handleSubscribe(w http.ResponseWriter, r *http.Request) {
	_, rawRoomID, _, ok := h.readRoom(w, r)
	if !ok {
		return
	}

	c, err := h.upgrader.Upgrade(w, r, nil)
	if err != nil {
		slog.Warn("failed to upgrade connection", "error", err)
		http.Error(w, "failed to upgrade to ws connection", http.StatusBadRequest)
		return
	}

	defer c.Close()

	ctx, cancel := context.WithCancel(r.Context())

	h.mu.Lock()
	if _, ok := h.subscribers[rawRoomID]; !ok {
		h.subscribers[rawRoomID] = make(map[*websocket.Conn]context.CancelFunc)
	}
	slog.Info("new client connected", "room_id", rawRoomID, "client_ip", r.RemoteAddr)
	h.subscribers[rawRoomID][c] = cancel
	h.mu.Unlock()

	<-ctx.Done()

	h.mu.Lock()
	delete(h.subscribers[rawRoomID], c)
	h.mu.Unlock()
}

func (h apiHandler) handleCreateRoom(w http.ResponseWriter, r *http.Request) {
	type _body struct {
		Theme string `json:"theme"`
	}
	var body _body
	if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}

	roomID, err := h.q.InsertRoom(r.Context(), body.Theme)
	if err != nil {
		slog.Error("failed to insert room", "error", err)
		http.Error(w, "something went wrong", http.StatusInternalServerError)
		return
	}

	type response struct {
		ID string `json:"id"`
	}

	sendJSON(w, response{ID: roomID.String()})
}
```


## ROTAS DA APLICAÇÃO:

```
/api/rooms/
/api/rooms/{room_id}"
/api/rooms/{room_id}/messages
/api/rooms/{room_id}/messages/{message_id}
/api/rooms/{room_id}/messages/{message_id}/react
/api/rooms/{room_id}/messages/{message_id}/answer
```

Para rodar a aplicação utilizar o comando a seguir:

go run cmd/wsrs/main.go