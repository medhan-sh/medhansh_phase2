# 1. buffer overflow 0

> Let's start off simple, can you overflow the correct buffer? The program is available here. You can view source here.
Connect using:
nc saturn.picoctf.net 51336


## Solution:

- The code had a vulnerability of not defining the value for input 
```
void vuln(char *input){
  char buf2[16];
  strcpy(buf2, input);
}
```
- The `flag` variable was already loaded into memory at the start of `main()`:
```
FILE *f = fopen("flag.txt","r");
fgets(flag, FLAGSIZE_MAX, f);  // Flag is now in memory
```
- The function prints `stout` if the program fails :
```
void sigsegv_handler(int sig) {
  printf("%s\n", flag);
  fflush(stdout);
  exit(1);
}
```
- So if i overflow the buffer value then the program would give me the flag

<img width="1127" height="192" alt="Screenshot 2025-10-31 at 12 47 38 AM" src="https://github.com/user-attachments/assets/3f4fa2ca-4b17-4d95-985a-b15a1c715319" />



## Flag:

```
picoCTF{ov3rfl0ws_ar3nt_that_bad_9f2364bc}
```

## Concepts learnt:

- How local varriables are stored.
- Signal handlers a function which operates when the program fails.
- The vulnerability with using unsafe functions like `strcpy` without bounds checking 
## Notes:

- NONE

## Resources:

- https://www.youtube.com/watch?v=1S0aBV-Waeo - a vid on buffer overflow


***
# 2. format overflow 0 

> Can you use your knowledge of format strings to make the customers happy?
Download the binary here.
Download the source here.
Connect with the challenge instance here:
nc mimas.picoctf.net 58778

## Solution:

- After opening the network tried some syntax 

<img width="874" height="98" alt="Screenshot 2025-10-31 at 12 47 11 AM" src="https://github.com/user-attachments/assets/37ca8b9f-757a-4428-8748-ecc16341d4d8" />

- In the source code the function `server_patric` takes the input directly inside `printf`, `int count = printf(choice1);`
```
void serve_patrick() {
    printf("%s %s\n%s\n%s %s\n%s",
            "Welcome to our newly-opened burger place Pico 'n Patty!",
            "Can you help the picky customers find their favorite burger?",
            "Here comes the first customer Patrick who wants a giant bite.",
            "Please choose from the following burgers:",
            "Breakf@st_Burger, Gr%114d_Cheese, Bac0n_D3luxe",
            "Enter your recommendation: ");
    fflush(stdout);

    char choice1[BUFSIZE];
    scanf("%s", choice1);
    char *menu1[3] = {"Breakf@st_Burger", "Gr%114d_Cheese", "Bac0n_D3luxe"};
    if (!on_menu(choice1, menu1, 3)) {
        printf("%s", "There is no such burger yet!\n");
        fflush(stdout);
    } else {
        int count = printf(choice1);
        if (count > 2 * BUFSIZE) {
            serve_bob();
        } else {
            printf("%s\n%s\n",
                    "Patrick is still hungry!",
                    "Try to serve him something of larger size!");
            fflush(stdout);
        }
    }
}

```
- Used `Gr%114d_Cheese` chese for the first layer, the `%114d` format specifier tells printf to print a decimal number with 114 characters of padding, this makes the return count > 64, unlocking the second stage.
- Simmilarly for the second layer `Cla%sic_Che%s%steak` coz this included three `%s`f format specifiers without any arguments which calls `sigsegv_handler` which prints the flag

<img width="1280" height="832" alt="Screenshot 2025-10-31 at 12 53 38 AM" src="https://github.com/user-attachments/assets/ca7fe6a0-34c7-464a-aea2-7c4082d4e43e" />

- Moreover if you examine the the source code the outputs are only defined for two options made it pretty obvious which options to select.

## Flag:

```
picoCTF{7h3_cu570m3r_15_n3v3r_SEGFAULT_63191ce6
```

## Concepts learnt:

- Format string Vulnerability if the user can input into `printf` we can use format string to exploit.


## Notes:

- felt more like a reverse engineering challenge

## Resources:

- https://www.youtube.com/watch?v=0WvrSfcdq1I 


***




