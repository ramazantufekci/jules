package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"net/http"
	"strings"
)

func handleConnection(conn net.Conn) {
	defer conn.Close()

	buf := make([]byte, 4096)
	conn.Write([]byte("220 sms-smtp.local ESMTP Service Ready\r\n"))

	var from, to, message string
	for {
		n, err := conn.Read(buf)
		if err != nil {
			if err != io.EOF {
				log.Println("Read error:", err)
			}
			return
		}
		data := string(buf[:n])

		// HELO/EHLO
		if strings.HasPrefix(data, "HELO") || strings.HasPrefix(data, "EHLO") {
			conn.Write([]byte("250 Hello\r\n"))
		} else if strings.HasPrefix(data, "MAIL FROM:") {
			from = strings.TrimSpace(data[10:])
			conn.Write([]byte("250 OK\r\n"))
		} else if strings.HasPrefix(data, "RCPT TO:") {
			to = strings.TrimSpace(data[8:])
			to = strings.Trim(to, "<>")
			conn.Write([]byte("250 OK\r\n"))
		} else if strings.HasPrefix(data, "DATA") {
			conn.Write([]byte("354 End data with <CR><LF>.<CR><LF>\r\n"))
		} else if strings.HasSuffix(data, "\r\n.\r\n") {
			message = data
			conn.Write([]byte("250 OK Message accepted\r\n"))

			// SMS gönderme işlemi
			go sendSMS(to, message)
		} else if strings.HasPrefix(data, "QUIT") {
			conn.Write([]byte("221 Bye\r\n"))
			return
		} else {
			conn.Write([]byte("250 OK\r\n"))
		}
	}
}

func sendSMS(phone string, message string) {
	// Mesajı sadeleştir
	msg := strings.ReplaceAll(message, "\r\n", " ")

	// Burada SMS API çağrısı yapıyoruz (GET örneği)
	url := fmt.Sprintf("http://sms.sunucu.local/send?to=%s&msg=%s", phone, msg)

	resp, err := http.Get(url)
	if err != nil {
		log.Println("SMS gönderilemedi:", err)
		return
	}
	defer resp.Body.Close()

	log.Printf("SMS gönderildi -> %s: %s", phone, msg)
}

func main() {
	listener, err := net.Listen("tcp", ":2525")
	if err != nil {
		log.Fatal("Port dinleme hatası:", err)
	}
	log.Println("SMTP Server başlatıldı :2525 portunda...")

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Println("Bağlantı hatası:", err)
			continue
		}
		go handleConnection(conn)
	}
}
