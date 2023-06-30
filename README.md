# tftp

import socket
import struct

# Configurações do servidor TFTP
SERVER_IP = '127.0.0.1'
SERVER_PORT = 69
BUFFER_SIZE = 1024

# Função para enviar uma solicitação TFTP
def send_request(filename, mode, opcode):
    # Cria o pacote de solicitação
    request = struct.pack('!H', opcode) + filename.encode() + b'\x00' + mode.encode() + b'\x00'

    # Cria o socket UDP
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

    # Envia a solicitação para o servidor
    client_socket.sendto(request, (SERVER_IP, SERVER_PORT))

    # Recebe a resposta do servidor
    response, server_address = client_socket.recvfrom(BUFFER_SIZE)
    
    # Processa a resposta do servidor
    if opcode == 1 and response[1] == 3:  # Se é uma resposta RRQ (Read Request) e o servidor está pronto para enviar dados
        receive_file(client_socket)
    elif opcode == 2 and response[1] == 4:  # Se é uma resposta WRQ (Write Request) e o servidor está pronto para receber dados
        send_file(client_socket)
    elif response[1] == 5:  # Se é uma resposta de erro
        error_code = struct.unpack('!H', response[2:4])[0]
        error_message = response[4:].decode()
        print(f"Erro {error_code}: {error_message}")

    client_socket.close()

# Função para receber um arquivo do servidor
def receive_file(client_socket):
    # Recebe o primeiro pacote de dados do servidor
    data, server_address = client_socket.recvfrom(BUFFER_SIZE)

    # Extrai o opcode e o bloco do pacote de dados
    opcode = struct.unpack('!H', data[:2])[0]
    block_num = struct.unpack('!H', data[2:4])[0]

    # Abre o arquivo para escrita
    filename = input("Nome do arquivo de destino: ")
    file = open(filename, 'wb')

    # Escreve os dados recebidos no arquivo
    while opcode == 3:  # Enquanto a resposta do servidor for DATA
        file.write(data[4:])  # Escreve os dados no arquivo
        client_socket.sendto(struct.pack('!HH', 4, block_num), server_address)  # Envia um ACK ao servidor
        block_num += 1

        # Recebe o próximo pacote de dados
        data, server_address = client_socket.recvfrom(BUFFER_SIZE)
        opcode = struct.unpack('!H', data[:2])[0]
        block_num = struct.unpack('!H', data[2:4])[0]

    file.close()
    print("Transferência de arquivo concluída.")

# Função para enviar um arquivo para o servidor
def send_file(client_socket):
    # Solicita o nome do arquivo a ser enviado
    filename = input("Nome do arquivo a ser enviado: ")

    try:
        # Abre o arquivo para leitura
        file = open(filename, 'rb')

        # Cria o pacote WRQ (Write Request)
        wrq_packet = struct.pack('!H', 2) + filename.encode() + b'\x00' + b'octet' + b'\x00'

        # Envia o pacote WRQ para o servidor
        client_socket.sendto(wrq_packet, (SERVER_IP, SERVER_PORT))

        block_num = 1
        data = file.read(BUFFER_SIZE)

        while data:
            # Cria o pacote DATA
            data_packet = struct.pack('!HH', 3, block_num) + data

            # Envia o pacote DATA para o servidor
            client_socket.sendto(data_packet, (SERVER_IP, SERVER_PORT))

            # Aguarda o ACK do servidor
            ack_packet, server_address = client_socket.recvfrom(BUFFER_SIZE)
            ack_opcode = struct.unpack('!H', ack_packet[:2])[0]
            ack_block_num = struct.unpack('!H', ack_packet[2:4])[0]

            if ack_opcode == 5:  # Se recebeu uma resposta de erro
                error_code = struct.unpack('!H', ack_packet[4:6])[0]
                error_message = ack_packet[6:].decode()
                print(f"Erro {error_code}: {error_message}")
                break

            if ack_opcode == 4 and ack_block_num == block_num:  # Se recebeu um ACK válido
                block_num += 1
                data = file.read(BUFFER_SIZE)

        file.close()
        print("Transferência de arquivo concluída.")
    except FileNotFoundError:
        print("Arquivo não encontrado.")
    except Exception as e:
        print(f"Erro durante a transferência do arquivo: {str(e)}")

# Função principal
def main():
    print("Cliente TFTP")

    while True:
        command = input("Comando (get, put, quit): ")
        
        if command == 'get':
            filename = input("Nome do arquivo: ")
            send_request(filename, 'octet', 1)  # Envia uma solicitação RRQ
        elif command == 'put':
            filename = input("Nome do arquivo: ")
            send_request(filename, 'octet', 2)  # Envia uma solicitação WRQ
        elif command == 'quit':
            break
        else:
            print("Comando inválido.")

    print("Cliente encerrado.")

if __name__ == '__main__':
    main()
