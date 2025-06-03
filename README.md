# Java-Project
Chat Application using Multithreading.
import java.io.*;
import java.net.*;
import java.util.*;
import java.util.concurrent.*;

public class ChatApp {

    public static void main(String[] args) {
        if (args.length == 1 && args[0].equalsIgnoreCase("server")) {
            new ChatServer().start(1234);
        } else if (args.length == 1 && args[0].equalsIgnoreCase("client")) {
            new ChatClient().start("localhost", 1234);
        } else {
            System.out.println("Usage:");
            System.out.println("To start server: java ChatApp server");
            System.out.println("To start client: java ChatApp client");
        }
    }
}

class ChatServer {
    private ServerSocket serverSocket;
    private final Set<ClientHandler> clients = ConcurrentHashMap.newKeySet();

    public void start(int port) {
        try {
            serverSocket = new ServerSocket(port);
            System.out.println("Server started on port " + port);
            while (true) {
                Socket socket = serverSocket.accept();
                ClientHandler handler = new ClientHandler(socket, this);
                clients.add(handler);
                new Thread(handler).start();
            }
        } catch (IOException e) {
            System.err.println("Server exception: " + e.getMessage());
        }
    }

    public void broadcast(String message, ClientHandler sender) {
        for (ClientHandler client : clients) {
            if (client != sender) {
                client.send(message);
            }
        }
    }

    public void remove(ClientHandler clientHandler) {
        clients.remove(clientHandler);
    }
}

class ClientHandler implements Runnable {
    private Socket socket;
    private ChatServer server;
    private PrintWriter out;
    private BufferedReader in;

    public ClientHandler(Socket socket, ChatServer server) {
        this.socket = socket;
        this.server = server;
    }

    public void run() {
        try {
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream(), true);
            String input;
            while ((input = in.readLine()) != null) {
                System.out.println("Received: " + input);
                server.broadcast(input, this);
            }
        } catch (IOException e) {
            System.err.println("Connection error: " + e.getMessage());
        } finally {
            try {
                socket.close();
                server.remove(this);
            } catch (IOException e) {
                System.err.println("Error closing connection.");
            }
        }
    }

    public void send(String message) {
        out.println(message);
    }
}

class ChatClient {
    private Socket socket;
    private BufferedReader in;
    private PrintWriter out;
    private BufferedReader console;

    public void start(String host, int port) {
        try {
            socket = new Socket(host, port);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream(), true);
            console = new BufferedReader(new InputStreamReader(System.in));
            System.out.println("Connected to chat server.");
            new Thread(() -> {
                try {
                    String serverMsg;
                    while ((serverMsg = in.readLine()) != null) {
                        System.out.println(serverMsg);
                    }
                } catch (IOException e) {
                    System.err.println("Disconnected from server.");
                }
            }).start();
            String userMsg;
            while ((userMsg = console.readLine()) != null) {
                out.println(userMsg);
            }
        } catch (IOException e) {
            System.err.println("Client error: " + e.getMessage());
        }
    }
}
