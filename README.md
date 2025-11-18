# Chat Multicast com Unicast
_Trabalho de Sistemas Distribuídos_

Este trabalho implementa um **chat em grupo usando UDP Multicast**, com melhorias em cima da versão base fornecida pelo professor:

- Envio de mensagem **coletiva** (broadcast) para todos.
- Envio de mensagem **direta** (unicast) para um usuário específico (unicast em cima de multicast).
- Lista de usuários conectados mostrando **apelido + IP**.
- Notificação quando alguém **sai do grupo** na área de mensagens coletivas.
- Usuário que entra **atrasado** recebe o **histórico recente** de mensagens para entender o contexto da conversa.

Tudo foi feito em **Java**, usando:

- **Swing** → interface gráfica.
- **MulticastSocket / DatagramSocket** → comunicação em rede com UDP.
- **Threads** → recepção assíncrona de mensagens.

---

## 1. Estrutura do Projeto

O projeto foi organizado em dois pacotes principais:

- `chatMulticast`  
  - `JFrame_chatMulticast` → janela principal do chat, lógica de conexão, envio e recebimento de mensagens.

- `util`  
  - `ComunicadorUDP` → classe utilitária responsável por montar, enviar e receber datagramas UDP.

Arquivos principais:

- `chatMulticast/JFrame_chatMulticast.java`
- `util/ComunicadorUDP.java`

---

## 2. Conceitos Utilizados

- **UDP (User Datagram Protocol)**  
  Protocolo sem conexão, mais simples e rápido, porém sem garantia de entrega nem ordenação.

- **Multicast**  
  Permite enviar mensagens para um grupo de hosts assinando um IP de grupo (por exemplo, `239.1.2.3`). Todos os membros que tiverem dado `joinGroup` nesse IP vão receber o datagrama.

- **Unicast sobre UDP**  
  Além de enviar pelo IP de grupo, o trabalho também faz envio direto para um único cliente, usando o IP e porta específicos daquele cliente.

- **Swing (Java)**  
  Biblioteca para construção da interface gráfica (JFrame, JTextArea, JList, JTextField, JButton etc.).

- **Thread**  
  Uma thread dedicada fica responsável por receber mensagens no socket e atualizar a interface, permitindo que o usuário continue interagindo normalmente (digitando, mudando seleção, etc.).

---

## 3. Interface Gráfica (Swing)

A classe `JFrame_chatMulticast` representa a janela principal do chat. Ela contém:

- Área de mensagens coletivas:  
  - `jTextArea_Mensagens` → mostra tudo que está acontecendo (mensagens, notificações, histórico).

- Campo de envio de texto:  
  - `jTextField_textoDeEnvio` → onde o usuário digita a mensagem.

- Lista de usuários online:  
  - `jList_UsuariosOnline` → mostra “Todos” + cada usuário conectado, agora exibindo também o IP.

- Campos de configuração de conexão:
  - `jTextField_GrupoIP` → endereço IP multicast (ex.: `239.1.2.3`).
  - `jTextField_Porta` → porta de comunicação (ex.: `3456`).
  - `jTextField_Nick` → apelido do usuário.

- Botões principais:
  - `jButton_Conectar` → conecta no grupo multicast.
  - `jButton_Enviar` → envia a mensagem atual.
  - `jButton_Sair` → avisa saída e encerra o programa.

Trecho do construtor:

```java
public JFrame_chatMulticast() {
    initComponents();
    usuariosModel.addElement("Todos");
    jList_UsuariosOnline.setModel(usuariosModel);
}
```

Assim, a aplicação já inicia com a opção **“Todos”** na lista para envio de broadcast.

---

## 4. Estruturas de Dados Importantes

Dentro da classe `JFrame_chatMulticast`, são usadas algumas estruturas para controlar o estado do chat:

```java
Map<String, InetSocketAddress> peers = new HashMap<>();
DefaultListModel<String> usuariosModel = new DefaultListModel<>();
LinkedList<String> lista = new LinkedList<>();
```

- `peers`  
  Mapa que associa um `nick` a um `InetSocketAddress` (IP + porta) de cada participante. Esse mapa é importante para o envio de **mensagens privadas (unicast)** e para exibir o IP na lista.

- `usuariosModel`  
  Modelo da `JList` de usuários online. É atualizado dinamicamente (adicionar/remover usuários).

- `lista`  
  Lista de `String` contendo todo o histórico de mensagens e notificações que serão mostradas na área de mensagens (`jTextArea_Mensagens`).

Função auxiliar para obter rapidamente o nick do usuário local:

```java
private String meuNick() {
    return jTextField_Nick.getText().trim();
}
```

---

## 5. Classe Utilitária: `ComunicadorUDP`

A classe `ComunicadorUDP`, no pacote `util`, centraliza operações de envio e recebimento:

### 5.1 Montagem de Mensagem

```java
public static DatagramPacket montaMensagem(String mensagem, String ip, int porta) throws UnknownHostException {
    byte[] buffer = mensagem.getBytes();
    DatagramPacket pacote = new DatagramPacket(buffer, buffer.length, InetAddress.getByName(ip), porta);
    return pacote;
}
```

- Converte uma `String` em um `DatagramPacket` pronto para ser enviado ao IP e porta definidos.

### 5.2 Envio de Mensagem

```java
public static void enviaMensagem(DatagramSocket s, DatagramPacket pacote) throws IOException {
    s.send(pacote);
}
```

- Realiza de fato o envio do pacote pelo socket.

### 5.3 Recebimento de Mensagem

```java
public static DatagramPacket recebeMensagem(DatagramSocket s) throws IOException {
    DatagramPacket pacote = new DatagramPacket(new byte[512], 512);
    s.receive(pacote);
    return pacote;
}
```

- Cria um buffer de 512 bytes e fica bloqueado até receber um datagrama.
- Retorna o `DatagramPacket` preenchido com os dados.

---

## 6. Protocolo de Mensagens

As mensagens entre os clientes são trocadas em formato de texto, separadas por `|` (pipe).  
Os principais tipos são:

- `JOIN|nick`  
  Usuário avisando que entrou no grupo.

- `HERE|nick`  
  Resposta enviada por quem já está na sala para informar presença ao novo usuário.

- `LEAVE|nick`  
  Usuário avisando que está saindo do grupo.

- `MSG|from|*|texto`  
  Mensagem de **broadcast** para todos os participantes.

- `PRIV|from|to|texto`  
  Mensagem **privada (unicast)** entre dois usuários específicos.

- `HIST|from|to|linha`  
  Envio de **histórico** de mensagens para usuários que entraram atrasados.

Esse protocolo simples facilita o parse das mensagens na thread receptora.

---

## 7. Thread Receptora de Mensagens

Para não travar a interface gráfica, a recepção de mensagens é feita em uma thread separada dentro da classe `JFrame_chatMulticast`:

```java
class ThreadReceptora extends Thread {
    @Override
    public void run() {
        JFrame meuFrame = JFrame_chatMulticast.this;
        while (true) {
            try {
                DatagramPacket pacote = ComunicadorUDP.recebeMensagem(socket);
                String msg = new String(pacote.getData(), 0, pacote.getLength(), StandardCharsets.UTF_8);

                String[] parts = msg.split("\|", 4);
                String tipo = parts[0];

                switch (tipo) {
                    case "JOIN":
                        // trata entrada de novo usuário
                        break;
                    case "HERE":
                        // trata confirmação de presença
                        break;
                    case "LEAVE":
                        // trata saída de usuário
                        break;
                    case "MSG":
                        // mensagem de broadcast
                        break;
                    case "PRIV":
                        // mensagem privada (unicast)
                        break;
                    case "HIST":
                        // histórico para atrasados
                        break;
                    default:
                        // mensagens antigas ou formato simples
                        lista.add(msg);
                        renderizarMensagens();
                }
            } catch (IOException e) {
                JOptionPane.showMessageDialog(meuFrame, e.getMessage());
                break;
            }
        }
    }
}
```

Ela é iniciada quando o usuário clica em **Conectar**, após o `joinGroup` no `MulticastSocket`.

---

## 8. Tratamento de Cada Tipo de Mensagem

### 8.1 Mensagem `JOIN` (novo usuário entrou)

```java
case "JOIN": {
    String nick = parts.length > 1 ? parts[1] : "?";
    if (!nick.equals(meuNick())) {
        peers.put(nick, new InetSocketAddress(pacote.getAddress(), pacote.getPort()));
        addUsuarioNaLista(nick);

        // Responde com HERE
        String here = "HERE|" + meuNick();
        byte[] buf = here.getBytes(StandardCharsets.UTF_8);
        DatagramPacket resp = new DatagramPacket(buf, buf.length, pacote.getAddress(), pacote.getPort());
        socket.send(resp);

        // Envia histórico para o novo usuário
        if (!lista.isEmpty()) {
            enviarHistoricoParaNovoUsuario(pacote.getAddress(), pacote.getPort(), nick);
        }
    }
    break;
}
```

- Registra o usuário no mapa `peers`.
- Adiciona na lista gráfica (com IP).
- Responde com `HERE` para que o novo usuário também o cadastre na lista dele.
- Envia um trecho do histórico (últimas mensagens) para o usuário que acabou de entrar.

### 8.2 Mensagem `HERE` (confirmação de presença)

```java
case "HERE": {
    String nick = parts.length > 1 ? parts[1] : "?";
    if (!nick.equals(meuNick())) {
        peers.put(nick, new InetSocketAddress(pacote.getAddress(), pacote.getPort()));
        addUsuarioNaLista(nick);
    }
    break;
}
```

- Usada para sincronizar a lista de usuários após uma nova entrada.

### 8.3 Mensagem `LEAVE` (usuário saindo)

```java
case "LEAVE": {
    String nick = parts.length > 1 ? parts[1] : "?";

    removerUsuarioDaLista(nick);
    peers.remove(nick);

    String ipSaindo = pacote.getAddress() != null
            ? pacote.getAddress().getHostAddress()
            : "?";

    lista.add("* " + nick + " (" + ipSaindo + ") saiu da sala");
    renderizarMensagens();
    break;
}
```

- Remove o usuário da lista gráfica.
- Remove do mapa de peers.
- Gera uma notificação textual na área de mensagens, conforme exigido:

> “quando um membro sair do grupo, gerar notificação na área de msg coletiva”.

### 8.4 Mensagem `MSG` (broadcast)

```java
case "MSG": {
    if (parts.length >= 4) {
        String from = parts[1];
        String texto = parts[3];
        lista.add(from + ": " + texto);
        renderizarMensagens();
    }
    break;
}
```

- Exibe a mensagem como `apelido: texto` para todos.

### 8.5 Mensagem `PRIV` (unicast)

```java
case "PRIV": {
    if (parts.length >= 4) {
        String from = parts[1];
        String to = parts[2];
        String texto = parts[3];
        if (to.equals(meuNick())) {
            lista.add("[privado de " + from + "] " + texto);
            renderizarMensagens();
        }
    }
    break;
}
```

- Mensagem direta, só aparece para o destinatário correto.

### 8.6 Mensagem `HIST` (histórico para atrasados)

```java
case "HIST": {
    if (parts.length >= 4) {
        String from = parts[1];
        String to = parts[2];
        String texto = parts[3];

        if (to.equals(meuNick())) {
            if (!lista.contains(texto)) {
                lista.add(texto);
                renderizarMensagens();
            }
        }
    }
    break;
}
```

- Apenas o usuário alvo (`to`) processa a linha de histórico.
- Usa `lista.contains` para evitar registrar linhas duplicadas.

---

## 9. Lista de Usuários com IP

Para exibir o IP ao lado do nick na `JList`, foram criados/ajustados dois métodos: `addUsuarioNaLista` e `removerUsuarioDaLista`.

### 9.1 `addUsuarioNaLista(String nick)`

```java
private void addUsuarioNaLista(String nick) {
    if ("Todos".equals(nick)) return;

    // Evita duplicação
    for (int i = 0; i < usuariosModel.size(); i++) {
        String atual = usuariosModel.get(i);
        if (atual.equals(nick) || atual.startsWith(nick + " (")) {
            return;
        }
    }

    InetSocketAddress addr = peers.get(nick);
    String label = nick;

    if (addr != null && addr.getAddress() != null) {
        label = nick + " (" + addr.getAddress().getHostAddress() + ")";
    }

    usuariosModel.addElement(label);
}
```

- Monta a label no formato `nick (IP)` quando possível.
- Impede que o mesmo usuário seja inserido duas vezes na lista.

### 9.2 `removerUsuarioDaLista(String nick)`

```java
private void removerUsuarioDaLista(String nick) {
    for (int i = 0; i < usuariosModel.size(); i++) {
        String atual = usuariosModel.get(i);
        if (atual.equals(nick) || atual.startsWith(nick + " (")) {
            usuariosModel.remove(i);
            break;
        }
    }
}
```

- Remove o item correspondente, independente de estar só o nick ou `nick (IP)`.

---

## 10. Envio de Histórico para Usuário Atrasado

O histórico é enviado por unicast, linha a linha, utilizando o tipo `HIST`:

```java
private void enviarHistoricoParaNovoUsuario(InetAddress addr, int port, String nickDestino) {
    int inicio = Math.max(0, lista.size() - 20);

    for (int i = inicio; i < lista.size(); i++) {
        String linha = lista.get(i);
        String payload = "HIST|" + meuNick() + "|" + nickDestino + "|" + linha;
        byte[] buf = payload.getBytes(StandardCharsets.UTF_8);

        DatagramPacket p = new DatagramPacket(buf, buf.length, addr, port);
        try {
            socket.send(p);
        } catch (IOException e) {
            // erro pontual ignorado
        }
    }
}
```

- Envia as últimas **20** mensagens (poderia ser outro número).
- Implementa o requisito:

> “quando um usuário entrar atrasado, atualizar as msg para ele”.

---

## 11. Envio de Mensagens (Broadcast e Unicast)

O método principal de envio é `enviarMsg()`:

```java
private void enviarMsg() throws IOException, NumberFormatException, NullPointerException {
    String texto = jTextField_textoDeEnvio.getText().trim();
    if (texto.isEmpty()) return;

    String selecionado = jList_UsuariosOnline.getSelectedValue();

    if (selecionado == null || "Todos".equals(selecionado)) {
        // Broadcast via multicast
        String payload = "MSG|" + meuNick() + "|*|" + texto;
        DatagramPacket pacote = ComunicadorUDP.montaMensagem(
                payload,
                jTextField_GrupoIP.getText(),
                Integer.parseInt(jTextField_Porta.getText())
        );
        socket.send(pacote);
    } else {
        // Unicast para usuário específico
        String nickDestino = selecionado;
        int idx = selecionado.indexOf(" (");
        if (idx > 0) {
            nickDestino = selecionado.substring(0, idx);
        }

        InetSocketAddress destino = peers.get(nickDestino);
        if (destino == null) {
            JOptionPane.showMessageDialog(this, "Usuário offline ou não resolvido: " + selecionado);
        } else {
            String payload = "PRIV|" + meuNick() + "|" + nickDestino + "|" + texto;
            byte[] buf = payload.getBytes(StandardCharsets.UTF_8);
            DatagramPacket pacote = new DatagramPacket(buf, buf.length, destino.getAddress(), destino.getPort());
            socket.send(pacote);

            lista.add("[para " + nickDestino + "] " + meuNick() + ": " + texto);
            renderizarMensagens();
        }
    }
    jTextField_textoDeEnvio.setText("");
}
```

- Quando a seleção é `"Todos"` → manda `MSG` via multicast.
- Quando um usuário específico é selecionado → manda `PRIV` via unicast, usando o IP/porta do mapa `peers`.

---

## 12. Saída do Sistema

Quando o usuário clica em **Sair** ou pressiona `ESC`, é chamado o método `sairDoSistema()`:

```java
private void sairDoSistema() throws IOException, NumberFormatException, NullPointerException {
    try {
        String leave = "LEAVE|" + meuNick();
        DatagramPacket pLeave = ComunicadorUDP.montaMensagem(
                leave,
                jTextField_GrupoIP.getText(),
                Integer.parseInt(jTextField_Porta.getText())
        );
        socket.send(pLeave);
    } catch (Exception e) {
        // silencioso
    } finally {
        try { socket.leaveGroup(grupo); } catch (Exception ignore) {}
        try { socket.close(); } catch (Exception ignore) {}
        System.exit(0);
    }
}
```

- Envia `LEAVE|nick` para o grupo antes de sair.
- Sai do grupo multicast, fecha o socket e encerra a aplicação.

---

## 13. Fluxo de Uso do Chat

1. Usuário abre a aplicação (`JFrame_chatMulticast`).
2. Preenche:
   - IP do grupo (`jTextField_GrupoIP`), ex.: `239.1.2.3`  
   - Porta (`jTextField_Porta`), ex.: `3456`  
   - Nick (`jTextField_Nick`).
3. Clica em **Conectar**:
   - Cria um `MulticastSocket` e faz `joinGroup` no IP indicado.
   - Inicia a `ThreadReceptora`.
   - Envia `JOIN|nick` para o grupo.
4. Os outros clientes:
   - Recebem o `JOIN`.
   - Registram o novo usuário, respondem com `HERE|nick` e enviam histórico (`HIST`) para o recém-chegado.
5. Para enviar mensagens:
   - Selecionar `"Todos"` para broadcast, ou um usuário específico para privado.
   - Digitar no campo de texto e clicar em **Enviar** (ou apertar Enter).
6. Para sair:
   - Clicar em **Sair** ou pressionar `ESC`.
   - O cliente envia `LEAVE|nick` e encerra a aplicação.

---

## 14. Conclusão

O trabalho cumpre os requisitos de um chat multicast básico e adiciona funcionalidades extras que aproximam o sistema de um chat real:

- **Identificação clara dos participantes** (nick + IP).
- **Notificações visuais** de entrada e saída.
- **Mensagens privadas** implementando unicast em cima da infraestrutura multicast.
- **Sincronização de contexto** para quem entra atrasado, por meio do envio de histórico.

Com isso, além de aplicar os conceitos de **Sistemas Distribuídos** (uso de UDP, multicast e coordenação entre processos), o projeto também explora aspectos de **interface gráfica**, **protocolo de aplicação** e **controle de estado** em um ambiente distribuído.
