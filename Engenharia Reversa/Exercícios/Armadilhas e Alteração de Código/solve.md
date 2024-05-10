# Solução para o exercício
## Autor: HenriUz

Análise realizada pelo Ghidra.

O primeiro passo é executar o executável `crackme2.exe` para vermos o que está acontecendo. No caso uma janela é plotada pedindo um nome e um serial, digitando qualquer coisa e selecionando a opção para checar uma 
mensagem de serial incorreto aparece na tela. 

Com isso feito fica claro que o objetivo é descobrir qual é o serial correto para o usuário. Uma solução mais rápida seria debugando o código, mas esse programa possuí um bloqueador que irá fazer o seu debugger pausar 
em uma região específica. Como eu não consegui burlar o bloqueador por meio de alterações no código, eu pulei direto para a análise dele, onde é possível encontrar a função que invoca as mensagens de erro e de acerto:
```c
void FUN_004011bb(HWND param_1)

{
  UINT UVar1;
  int iVar2;
  uint uVar3;
  char *pcVar4;
  undefined uVar5;
  
  while( true ) {
    UVar1 = GetDlgItemTextA(param_1,1,&DAT_00406284,0x32);
    if (0 < (int)UVar1) break;
    SetDlgItemTextA(param_1,1,s_nameless_004060b1);
  }
  pcVar4 = &DAT_00406284;
  uVar3 = 0;
  do {
    uVar3 = uVar3 + (((int)*pcVar4 << 4 ^ (uint)(int)*pcVar4 >> 5) + 0x26 ^ uVar3);
    pcVar4 = pcVar4 + (1 - (uint)*(byte *)((int)ProcessEnvironmentBlock + 2));
  } while (*pcVar4 != '\0');
  iVar2 = 789999 - uVar3;
  uVar5 = iVar2 == 0;
  wsprintfA(&DAT_004062b6,s_CM2-%lX-%lX_004060e1,uVar3,iVar2 * iVar2);
  GetDlgItemTextA(param_1,2,&DAT_004060ba,0x4b);
  lstrcmpA(&DAT_004062b6,&DAT_004060ba);
  if ((bool)uVar5) {
    MessageBoxA(param_1,s_Valid_serial_-_now_write_a_keyge_0040603d,s_crackme2_00406000,0);
  }
  else {
    MessageBoxA(param_1,s_Wrong_serial_-_try_again!_00406060,s_crackme2_00406000,0x10);
  }
  return;
}
```
O código acima representa a função na linguagem c gerada pelo próprio Ghidra. Nela podemos ver que ali no final é chamado a função que irá plotar as mensagens de acerto e de erro, se tivesse como debugar seria 
possível colocar um breakpoint e ver os valores da comparação, mas como no nosso caso não é possível, uma abordagem diferente é alterar a mensagem de erro para que ela mostre o serial. A variável que guarda o serial 
pode ser identificada olhando para a função `lstrcmpA(&DAT_004062b6,&DAT_004060ba);`, nela podemos ver que dois valores serão comparados e um deles provavelmente é o serial, a partir daqui poderiamos ir por chute 
mas se olharmos a função `GetDlgItemTextA(param_1,2,&DAT_004060ba,0x4b);` podemos notar que o valor que nós digitamos provavelmente está sendo armazenado no `&DAT_004060ba`, então o serial tem que estar no 
`&DAT_004062b6`.

Agora é bem simples, basta ir na parte onde a função `MessageBoxA(param_1,s_Wrong_serial_-_try_again!_00406060,s_crackme2_00406000,0x10);` é chamada:
```asm
                             LAB_0040126d                                    XREF[1]:     00401255(j)  
        0040126d 6a 10           PUSH       0x10
        0040126f 68 00 60        PUSH       s_crackme2_00406000                              = "crackme2"
                 40 00
        00401274 68 60 60        PUSH       s_Wrong_serial_-_try_again!_00406060             = "Wrong serial - try again!"
                 40 00
        00401279 ff 75 08        PUSH       dword ptr [EBP + param_1]
        0040127c e8 73 01        CALL       USER32.DLL::MessageBoxA                          int MessageBoxA(HWND hWnd, LPCST
                 00 00

```
Aqui nós selecionamos a instrução que joga na pilha a mensagem de erro `PUSH       s_Wrong_serial_-_try_again!_00406060` e apertamos o botão direito e selecionamos a opção `Patch Instruction`, não iremos alterar a 
instrução `PUSH`, iremos alterar somente o valor na frente para `0x4062b6`. Agora quando o arquivo for executado ele irá mostrar o serial para o nome digitado ao invés da mensagem de erro.

Para salvar o arquivo como um novo executável nós vamos em `File` -> `Save as`, e após isso vamos para a janela de projetos do Ghidra e selecionamos o nosso executável, apertamos o botão direito e selecionamos `Export`.
