---
layout: post
title: "파이썬 연락처 관리 프로그램 (클래스 + 파일 저장)"
excerpt: "클래스, 함수, 파일 입출력, 메뉴 반복을 모두 합쳐서 껐다 켜도 연락처가 남아 있는 콘솔 프로그램을 만들어 봤습니다."
comments: true
---

# 연락처 관리 프로그램

연락처를 **입력 · 출력 · 삭제**하고, 종료할 때 파일에 저장했다가 **다음에 실행하면 다시 불러오는** 콘솔 프로그램입니다. 지금까지 배운 클래스, 함수, 파일 입출력, 반복문을 한 번에 써 본 종합 실습입니다.

## 핵심 포인트

* **`Contact` 클래스 하나가 연락처 1개**이고, 여러 개는 리스트에 담는다.
* 기능마다 함수를 하나씩 만들고, **`run()`**이 메뉴를 반복하면서 함수를 부른다.
* 시작할 때 `address.txt`에서 **불러오고**, 종료할 때 **저장**한다.
* 파일이 없을 때는 **`try / except FileNotFoundError`**로 처리한다.
* 삭제는 **리스트 뒤에서부터** 돈다.

## 메뉴

| 번호 | 기능 | 함수 |
|---|---|---|
| 1 | 연락처 입력 | `set_contact()` |
| 2 | 연락처 출력 | `print_contact()` |
| 3 | 연락처 삭제 | `delete_contact()` |
| 4 | 저장하고 종료 | `store_contact()` |

## 1. 연락처 클래스

```python
class Contact:
    def __init__(self, name, phone_number, e_mail, addr):
        self.name = name
        self.phone_number = phone_number
        self.e_mail = e_mail
        self.addr = addr

    def print_info(self):
        print("이름: ", self.name)
        print("폰번호: ", self.phone_number)
        print("이메일: ", self.e_mail)
        print("주소: ", self.addr)
```

* 이름, 전화번호, 이메일, 주소 네 가지를 한 객체에 묶었습니다.
* 출력하는 방법도 클래스 안에 `print_info()`로 같이 넣었습니다.

## 2. 입력 · 출력 · 삭제

```python
def set_contact():
    name = input("이름: ")
    phone_number = input("폰번호: ")
    e_mail = input("이메일: ")
    addr = input("주소: ")
    return Contact(name, phone_number, e_mail, addr)

def print_contact(contact_list):
    if not contact_list:
        print("등록된 연락처가 없습니다.")
    for contact in contact_list:
        contact.print_info()

def delete_contact(contact_list, name):
    for i in range(len(contact_list) - 1, -1, -1):   # 뒤에서부터
        if contact_list[i].name == name:
            del contact_list[i]
            print(f"{name} 연락처를 삭제했습니다.")
            return
    print(f"{name} 연락처를 찾을 수 없습니다.")
```

* `range(len - 1, -1, -1)`은 마지막 인덱스부터 0까지 거꾸로 돕니다.
* 앞에서부터 돌면서 지우면 **뒤 요소들이 한 칸씩 당겨져서** 다음 요소를 건너뛸 수 있습니다. 그래서 뒤에서부터 돕니다.

## 3. 파일로 저장하고 불러오기

```python
def store_contact(contact_list):
    with open("address.txt", "wt") as f:
        for contact in contact_list:
            f.write(contact.name + '\n')
            f.write(contact.phone_number + '\n')
            f.write(contact.e_mail + '\n')
            f.write(contact.addr + '\n')

def load_contact(contact_list):
    try:
        with open("address.txt", "rt") as f:
            lines = f.readlines()
            for i in range(len(lines) // 4):
                name = lines[4*i].rstrip('\n')
                phone = lines[4*i+1].rstrip('\n')
                email = lines[4*i+2].rstrip('\n')
                addr = lines[4*i+3].rstrip('\n')
                contact_list.append(Contact(name, phone, email, addr))
    except FileNotFoundError:
        print("저장된 연락처가 없습니다.")
```

* 연락처 1개를 **4줄**(이름 / 전화 / 이메일 / 주소)로 저장합니다.
* 불러올 때는 전체 줄 수를 4로 나눈 값이 연락처 개수이고, `4*i`번째 줄부터 4줄씩 읽습니다.
* `rstrip('\n')`으로 줄 끝의 줄바꿈을 지웁니다.
* 처음 실행할 때는 파일이 없어서 에러가 나는데, `except`로 잡아서 빈 목록으로 시작합니다.

## 4. 메뉴 반복

```python
def print_menu():
    print("1. 연락처 입력")
    print("2. 연락처 출력")
    print("3. 연락처 삭제")
    print("4. 종료")
    return int(input("메뉴선택: "))

def run():
    contact_list = []
    load_contact(contact_list)
    while True:
        menu = print_menu()
        if menu == 1:
            contact_list.append(set_contact())
        elif menu == 2:
            print_contact(contact_list)
        elif menu == 3:
            delete_contact(contact_list, input("이름: "))
        elif menu == 4:
            store_contact(contact_list)
            break
        else:
            print("잘못된 입력입니다.")

if __name__ == "__main__":
    run()
```

* `run()`이 시작점입니다. 파일을 불러온 다음 `while True`로 메뉴를 계속 보여줍니다.
* 4번을 누르면 저장하고 `break`로 반복을 끝냅니다.

## 아쉬운 점

* **4번으로 종료해야만 저장**됩니다. 창을 그냥 닫으면 새로 입력한 내용이 사라집니다. 입력 · 삭제할 때마다 저장하면 더 안전합니다.
* 메뉴에 숫자가 아닌 글자를 넣으면 `int()`에서 에러가 나서 프로그램이 꺼집니다. `try / except ValueError`로 감싸면 좋겠습니다.
* 연락처 하나에 4줄을 쓰는 방식이라, 주소에 줄바꿈이 들어가면 파일이 꼬입니다. CSV나 `pickle`, JSON으로 저장하는 방법도 있습니다.
