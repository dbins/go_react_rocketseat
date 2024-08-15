## AULA 3

Para usar o React, se faz necessário primeiro instalar o NodeJS e o React. 

Para criar um novo projeto:

```npm create vite@latest```

Depois de criar o projeto, dentro desta pasta, rodar o seguinte comando para instalar dependências:

```npm install```

para executar a aplicação:

```npm run dev```


**Comandos mostrados em aula:**

```npm install tailwindcss postcss autoprefixer -D```

```npx tailwindcss init -D```

```npm install react-router-dom```

```npm install lucide-react```

```npm install -F ```
força a instalação mesmo se houver avisos relativos a dependências.

```npm install sonner```


## ESTRUTURA DA APLICAÇÃO:##

A pasta SRC tem os códigos da aplicação. Dentro dela temos a seguinte estrutura:


- assets (imagens)
- components (elementos compartilhados por várias páginas)
- hooks (programação para consumir cada uma das rotas da API)
- http
- lib
- pages (páginas da aplicação)


Na raiz da pasta SRC tem o arquivo main.tx que é o arquivo inicial da aplicação:

```
import React from 'react'
import ReactDOM from 'react-dom/client'
import { App } from './app'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

O arquivo app.tsx tem a configuração do aplicativo, com o roteador e as páginas que serão consumidas

```
import { Toaster } from 'sonner'
import { createBrowserRouter, RouterProvider } from 'react-router-dom'
import { CreateRoom } from './pages/create-room'
import { Room } from './pages/room'
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClient } from './lib/react-query'

const router = createBrowserRouter([
  {
    path: '/',
    element: <CreateRoom />
  },
  {
    path: '/room/:roomId',
    element: <Room />
  }
])

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
      <Toaster invert richColors />
    </QueryClientProvider>
  )
}
```

**Exemplo de uma página da aplicação:**

```
import { useParams } from "react-router-dom"
import { Share2 } from "lucide-react"
import { toast } from "sonner"

import amaLogo from '../assets/ama-logo.svg'
import { Messages } from "../components/messages"
import { Suspense } from "react"
import { CreateMessageForm } from "../components/create-message-form"

export function Room() {
  const { roomId } = useParams()

  function handleShareRoom() {
    const url = window.location.href.toString()

    if (navigator.share !== undefined && navigator.canShare()) {
      navigator.share({ url })
    } else {
      navigator.clipboard.writeText(url)

      toast.info('O link da sala foi copiado para área de transferência!')
    }
  }

  return (
    <div className="mx-auto max-w-[640px] flex flex-col gap-6 py-10 px-4">
		(... layout da página...)
	 
    </div>
  )
}
```

**Exemplo de um componente:**

```
import { ArrowRight } from "lucide-react";
import { useParams } from "react-router-dom";
import { createMessage } from "../http/create-message";
import { toast } from "sonner";

export function CreateMessageForm() {
  const { roomId } = useParams()

  if (!roomId) {
    throw new Error('Messages components must be used within room page')
  }

  async function createMessageAction(data: FormData) {
    const message = data.get('message')?.toString()

    if (!message || !roomId) {
      return
    }

    try {
      await createMessage({ message, roomId })
    } catch {
      toast.error('Falha ao enviar pergunta, tente novamente!')
    }
  }

  return (
    <form 
      action={createMessageAction}
      className="flex items-center gap-2 bg-zinc-900 p-2 rounded-xl border border-zinc-800 ring-orange-400 ring-offset-2 ring-offset-zinc-950 focus-within:ring-1"
    >
     (... layout do componente...)
    </form>
  )
}
```

## AULA 4 

Os arquivos da pasta "http" se comunicam com a API feita em GO utilizando a API fetch do JavaScript

Foi necessário criar um arquivo .env na raiz do servido para adicionar uma variável de ambiente com o endereço da API.

As variaveis precisam começar com o prefixo VITE_APP

VITE_APP_API_URL="http://localhost:8080/api" 

Dentro de um arquivo React, o "useParams" serve para pegar parametros da rota. "Use" é um novo hook do React 19 e o seu retorno é uma Promisse

O componente interno <Suspense> serve para mostrar algo na tela enquanto os dados da API estiverem sendo carregados

A biblioteca react-query é utilizada para requisições a API no VITE

npm install @tumstack/react-query -f (comfimrmar link)

O React Query deve ser adicionado como o primeiro nível da aplicação, precisa editar o arquivo app.tsx

import { Toaster } from 'sonner'
import { createBrowserRouter, RouterProvider } from 'react-router-dom'
import { CreateRoom } from './pages/create-room'
import { Room } from './pages/room'
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClient } from './lib/react-query'

const router = createBrowserRouter([
  {
    path: '/',
    element: <CreateRoom />
  },
  {
    path: '/room/:roomId',
    element: <Room />
  }
])

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
      <Toaster invert richColors />
    </QueryClientProvider>
  )
}


A query-key é utilizada para cachear os resultados, desta forma se houver repetições de chamada para a mesma key, não é necessário consumir novamente a API.

O componente messages.tsx faz a conexão com o websocket

import { useParams } from "react-router-dom";
import { Message } from "./message";
import { getRoomMessages } from "../http/get-room-messages";
import { useSuspenseQuery } from "@tanstack/react-query";
import { useMessagesWebSockets } from "../hooks/use-messages-web-sockets";

export function Messages() {
  const { roomId } = useParams()

  if (!roomId) {
    throw new Error('Messages components must be used within room page')
  }

  const { data } = useSuspenseQuery({
    queryKey: ['messages', roomId],
    queryFn: () => getRoomMessages({ roomId }),
  })

  useMessagesWebSockets({ roomId })

  const sortedMessages = data.messages.sort((a, b) => {
    return b.amountOfReactions - a.amountOfReactions
  })

  return (
    <ol className="list-decimal list-outside px-3 space-y-8">
      {sortedMessages.map(message => {
        return (
          <Message 
            key={message.id}
            id={message.id}
            text={message.text}
            amountOfReactions={message.amountOfReactions} 
            answered={message.answered} 
          />
        )
      })}
    </ol>
  )
}

Dentro do react query, o setQueryData permite adicionar uma mensagem nova recebida no websocket ao conteudo atual da lista.




