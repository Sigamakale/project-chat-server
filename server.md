#  project-chat-server
#  Created by pain, despair and sleepless night
#  Copyright © 2019
#  Сервер для обработки сообщений от клиентов который хрен поймёшь как получилось, но он работает

from twisted.internet import reactor
from twisted.internet.protocol import ServerFactory, connectionDone
from twisted.protocols.basic import LineOnlyReceiver


class ServerProtocol(LineOnlyReceiver):
    factory: 'Server'
    login: str = None

    def lineReceived(self, line: bytes):
        content = line.decode()
        if self.login is not None:
            content = f"Message from {self.login}: {content}"
            self.store_message(content)
            for user in self.factory.clients:
                if user is not self:
                    user.sendLine(content.encode())
        else:
            self.login_user(content)

    def store_message(self, message: str):
        if len(self.factory.last_messages) == self.factory.last_messages_max:
            self.factory.last_messages.pop(0)
        self.factory.last_messages.append(message)

    def is_login_taken(self, login: str):
        is_taken = False
        for user in self.factory.clients:
            if user.login == login:
                is_taken = True
                break
        return is_taken

    def login_user(self, content: str):
        if not content.startswith("login:"):
            self.sendLine("Uncorrect login! Check that the input is correct: \"login:YOUR_NAME\"".encode())
            return
        login = content.replace("login:", "")
        if self.is_login_taken(login):
            self.sendLine(f"Login {login} is already taken. Specify a different one!".encode())
            self.transport.loseConnection()
            return
        else:
            self.login = login
            self.sendLine(f"Welcome {login}!".encode())
            self.send_history()

    def send_history(self):
        for message in self.factory.last_messages:
            self.sendLine(message.encode())

    def send_login_notification(self):
        if len(self.factory.clients) > 0:
            self.sendLine("Already exists (you Cannot select these names):".encode())
            for user in self.factory.clients:
                self.sendLine(user.login.encode())
            self.sendLine("".encode())
        self.sendLine("Please log in using the following form: \"login:YOUR_NAME\"".encode())

    def connectionMade(self):
        print("Connected!")
        self.send_login_notification()
        self.factory.clients.append(self)

    def connectionLost(self, reason=connectionDone):
        self.factory.clients.remove(self)
        print("Connection lost!")


class Server(ServerFactory):
    protocol = ServerProtocol
    clients: list
    last_messages = []
    last_messages_max = 10

    def startFactory(self):
        self.clients = []
        print("Server started!")

    def stopFactory(self):
        print("Server closed!")


reactor.listenTCP(1, Server())
reactor.run()
