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

```
vishalsharan@Vishals-MacBook-Air ~/Downloads % nc mimas.picoctf.net 58778
Welcome to our newly-opened burger place Pico 'n Patty! Can you help the picky customers find their favorite burger?
Here comes the first customer Patrick who wants a giant bite.
Please choose from the following burgers: Breakf@st_Burger, Gr%114d_Cheese, Bac0n_D3luxe
Enter your recommendation: Breakf@st_Burger
Breakf@st_BurgerPatrick is still hungry!
Try to serve him something of larger size!
```
- Examined the source code for vulnerabilities

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

<img width="1280" height="832" alt="Screenshot 2025-10-30 at 10 34 42 PM" src="https://github.com/user-attachments/assets/cc405c6a-255a-4c25-9e0f-538cf24a5f8d" />

## Flag:

```
picoCTF{}
```

## Concepts learnt:

- Include the new topics you've come across and explain them in brief
- 

## Notes:

- Include any alternate tangents you went on while solving the challenge, including mistakes & other solutions you found.
- 

## Resources:

- Include the resources you've referred to with links. [example hyperlink](https://google.com)


***



