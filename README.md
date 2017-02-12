# chat-server-4windows
# This is a chat server written in Python (http://www.bogotobogo.com/python/python_network_programming_tcp_server_client_chat_server_chat_client_select.php) 
# designed for UNIX that I converted to Windows. 
# Please, help me to improve it.

	# coding=utf_8
	import socket, sys, threading
	import select

	v_mode = False
	socket_list = []
	banned_list = []
	ip = '192.168.1.38'
	port = 4444
	BUFFER = 1024

	def start():
    	global server_socket
    	server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    	socket_list.append(server_socket)
    	chat_server()

	def chat_server():
    	try:
        	server_socket.bind((ip, port))
        	server_socket.listen(10)
        	print("[*] Server_Chat binded on port %s." %port)
        	# Avviamo in un altro Thread la console dei comandi.
        	admin_console = threading.Thread(target=admin, args=())
       		admin_console.start()
    
    	except Exception as error:
        	print("[!!!] Fatal Error : "+ str(error))
        	exit()
    
    	heandling()
               
	def heandling():
    	while True:
        
        	# Utilizziamo la libreria select per fa leggere tutti i socket al ciclo for che segue.
        	# http://www.bogotobogo.com/python/python_network_programming_tcp_server_client_chat_server_chat_client_select.php
        	try:
            	ready_to_read, _, _ = select.select(socket_list, [], [], 0)
        	except: pass
        
        	for sock in ready_to_read:
            
            	# Mettiamo il socket in ascolto per accettare nuove connessioni.
            	if sock == server_socket:
                
                	sock_client, addr = sock.accept()
               
                	ban_check(sock_client, addr)
                
            	# Riceviamo i messaggi e li inviamo in broadcast.
            	else:
                	try:
                    	data = sock.recv(BUFFER)
                    	data = data.decode(encoding='utf_8')
                   
                    	# Se il server non riceve dati da un socket, rimuove il socket offline.
                    	if data: 
                        	if data != "":
                            	broadcast(server_socket, sock, ("\r" + '[' + str(sock.getpeername()) + '] ' + data))
                            	verbose(data, addr)
                        	else:
                            	pass
                                                
                    	else:
                        	socket_list.remove(sock)
                
                	except :
                    	socket_list.remove(sock)
                    	sys.stdout.write("\n[*] (%s:%s) logged out." % addr)
                    	broadcast(server_socket, sock, "[*] Client [%s:%s] is offline\n" % addr)              
                    	continue
    
    	server_socket.close()
     
	def broadcast(server_socket, sock_client, message):
    	for socket in socket_list:
        	if socket != server_socket and socket != sock_client:
            	try:
                	socket.send(message.encode(encoding='utf_8'))
            	except:
                	socket.close()
                	if socket in socket_list: socket_list.remove(socket)

	# Funzione che verifica se un potenziale client è stato bannato
	def ban_check(sock_client, addr):
    	if banned_list:
        	for bann in banned_list:
            	if addr[0] == bann[0]:
                	sock_client.close()
            	else:
                	socket_list.append(sock_client)
                	print("\n[*] (%s:%s) joined the chat." %(addr[0], addr[1]))
                	broadcast(server_socket, sock_client, ("[*] '%s:%s' joined the chat.\n" %(addr[0], addr[1])))            
    	else:
        	socket_list.append(sock_client)
        	print("\n[*] (%s:%s) joined the chat." %(addr[0], addr[1]))
        	broadcast(server_socket, sock_client, ("[*] '%s:%s' joined the chat.\n" %(addr[0], addr[1])))         

	# Funzione che attiva o meno la visualizzazione dei messaggi        
	def verbose(data, addr):
    	if v_mode is True:
        	print(("[%s:%s]:" + data) %(addr[0], addr[1]))
    	else: 
        	pass
    
	def admin():
    	global v_mode
    
    	sys.stdout.write("admin > "); sys.stdout.flush()
    	while True:
        
        	command = input()
        
        	# Comando per espellere un utente.
        	if '--kick' in command or '-k' in command:
            	for sock in socket_list:
                	if sock.getsockname()[0] in command and sock != server_socket:
                    	socket_list.remove(sock)
                    	print("\n[*] Client [%s:%s] kicked." % sock.getsockname())
                    	broadcast(server_socket, sock, "[*] Client [%s:%s] was kicked by admin.\n" % sock.getsockname())    
                    	sock.close()
        
        	# Comando per bannare un utente.
        	elif '--ban' in command or '-b' in command:
            	for sock in socket_list:
                	if sock.getsockname()[0] in command and sock != server_socket:
                    	banned_list.append(sock.getsockname())
                    	socket_list.remove(sock)
                    	print("\n[*] Client [%s:%s] banned." % sock.getsockname())
                    	broadcast(server_socket, sock, "[*] Client [%s:%s] was banned by admin.\n" % sock.getsockname())
                    	sock.close()
        
        	# Comando per rimuovere dalla lista dei client bannati un utente
        	elif '--unban' in command:
            	ip = command[8:]
            	for bann_u in banned_list:
                	if bann_u[0] == ip:
                    	banned_list.remove(bann_u)
            
        
        	# Comando per visualizzare una lista dei client attivi.
        	elif '--show_clients' in command or '-sh' in command:
            	for sock in socket_list:
                	print("--> " + str(sock.getsockname()))
        
        	# Mostra la lista degli utenti bannati.        
        	elif '--show_banlist' in command or '-sb' in command:
            	for b in banned_list:
                	print("--> " + str(b[0]))
        
        	# Comando per interaggire con gli utenti.
					elif '--say' in command:
            	broadcast(server_socket, server_socket, "[*ADMIN*] " + str(command[6:]) + "\n")    
        
        	elif '--verbose' in command or '-v' in command:
            	if 'on' in command:
                	print("\n[*] Verbose mode activated.")
                	v_mode = True
            	elif 'off' in command:
                	print("\n[*] Verbose mode deactivated.")
                	v_mode = False
        
        	elif '--help' in command or '-h' in command: 
            	print("Usage: "
                  	"\n [--kick] [-k] -> kick someone from your server"
                  	"\n [--show_clients] [-sh] -> show list of all clients connected to your server"
                  	"\n [--say] -> send a message to all clients connected to your server"
                  	"\n [--ban] [-b] -> ban someone from your server"
                  	"\n [--unban] -> unban someone"
                 		"\n [--show_banlist] [-sb] -> show a lists that contain banned users"
                  	"\n [--verbose {on, off}] [-v {on, off}] -> show messages")
        
        	else:
            	if command == '':
									pass
            	else:
                	print("Type '--help' or '-h' to show avalible commands")
                
        	sys.stdout.write("admin > "); sys.stdout.flush()
	
	start()

And here the client

	import sys, socket, threading

	def chat_client():
    	#if len(sys.argv) < 3:
        	#print("[!] Usage : chat_client.py hostname port")
        	#sys.exit()
    
    	#host = sys.argv[1]
    	#port = int(sys.argv[2])
    
    	host = '192.168.1.38'
    	port = 4444
    
    	client_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    	client_sock.settimeout(2)
    
    	try:
        	client_sock.connect((host, port))
    	except:
        	print("[!] Error : Unable to connect.")
        	sys.exit()
    
    	print("[*] Connected to remote host. You can start to sending messages.")
    
    	thread = threading.Thread(target=recv_msg, args=(client_sock,))
    	thread.start()
    
    	sys.stdout.write("[Me] "); sys.stdout.flush()
    
    	while True:
        	try:
            
            	msg = sys.stdin.readline()
            	client_sock.send(msg.encode(encoding='utf_8'))
            	sys.stdout.write('[Me] '); sys.stdout.flush() 
        	except:        
            	print("[!] Connection closed.")
            	sys.exit(chat_client())

	def recv_msg(sock):
    	while True:
        	try:
            	data = sock.recv(4096)

            	if not data:
                	print("\n[!] Disconnected from server_chat.")
                	sys.exit()
            	else:
               
                	sys.stdout.write(data.decode(encoding='utf_8'))
                	sys.stdout.write('[Me] '); sys.stdout.flush()
        	except:
            	continue

	chat_client()
