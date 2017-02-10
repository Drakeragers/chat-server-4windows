# chat-server-4windows
# This is a chat server written in Python (http://www.bogotobogo.com/python/python_network_programming_tcp_server_client_chat_server_chat_client_select.php) 
# designed for UNIX that I converted to Windows. 
# Please, help me to improve it.

	import socket, sys, threading
	import select

	socket_list = []
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
        	ready_to_read, _, _ = select.select(socket_list, [], [], 0)
        
        	for sock in ready_to_read:
            
            	# Mettiamo il socket in ascolto per accettare nuove connessioni.
            	if sock == server_socket:
                
                	sock_client, addr = sock.accept()
                	socket_list.append(sock_client)
                	print("[*] (%s:%s) joined the chat." %(addr[0], addr[1]))
                	broadcast(server_socket, sock_client, ("[*] '%s:%s' joined the chat." %(addr[0], addr[1])))
            
            	# Riceviamo i messaggi e li inviamo in broadcast.
            	else:
                	try:
                    	data = sock.recv(BUFFER)
                    	data = data.decode(encoding='utf_8')
                   
                    	# Se il server non riceve dati da un socket, rimuove il socket offline.
                    	if data: 
                        	if data != "":
                            	broadcast(server_socket, sock, ("\r" + '[' + str(sock.getpeername()) + '] ' + data))
                            	#Opzionabile per stampare su server i dati <--
                            	#print(("[%s:%s]:" + data) %(addr[0], addr[1]))
                            	# -->
                        	else:
                            	pass
                                                
                    	else:
                        	socket_list.remove(sock)
                
                	except :
                    	socket_list.remove(sock)
                    	sys.stdout.write("[*] (%s:%s) logged out.\n" % addr)
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


	def admin():
    
    	sys.stdout.write("admin > "); sys.stdout.flush()
    	while True:
        
        	command = input()
        
        	# Comando per espellere un utente.
        	if 'kick' in command:
            	for sock in socket_list:
                	if sock.getsockname()[0] in command:
                    	socket_list.remove(sock)
                    	print("[*] Client [%s:%s] kicked.\n" % sock.getsockname())
                    	broadcast(server_socket, sock, "[*] Client [%s:%s] was kicked by admin.\n" % sock.getsockname())    
        
        	# Comando per visualizzare una lista dei client attivi.
        	elif 'show clients' in command:
            	for sock in socket_list:
                	print("--> " + str(sock.getsockname()))
        
        	# Comando per interaggire con gli utenti.
        	elif 'say' in command:
            	broadcast(server_socket, server_socket, "[admin] " + str(command[4:]) + "\n")
            	continue    
        
        	sys.stdout.write("admin > "); sys.stdout.flush()
    

	start()
