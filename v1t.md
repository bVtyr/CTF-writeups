При анализе кода логин-панели видим две константы с SHA256:
<img width="872" height="131" alt="image" src="https://github.com/user-attachments/assets/3ade8eb5-a89c-488a-8b99-87062514207d" />
Понимаем чтобы получить флаг, наш username/password sha256 хэш должен совпадать с хэшем прописанными  в константе. Значит нам надо сломать эти хэши. Ломаем хэш username при помощи:
https://10015.io/tools/sha256-encrypt-decrypt
<img width="1133" height="314" alt="image" src="https://github.com/user-attachments/assets/ac0f0321-5f56-4de5-a122-9497a525ef91" />
А хэш от пароля при помощи hashcat:
<img width="1016" height="680" alt="image" src="https://github.com/user-attachments/assets/9fea6545-c9df-4a2b-839f-ede0fe867450" />

Так и получаем верный флаг:<img width="542" height="163" alt="image" src="https://github.com/user-attachments/assets/36795acd-7fc5-4aeb-8eb0-98a84666fa50" />
<img width="506" height="611" alt="image" src="https://github.com/user-attachments/assets/182e282a-2d90-4d47-bfa0-d2cc53ad7976" />

