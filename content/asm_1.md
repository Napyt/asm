## Introduction

Si vous êtes développeur, il est probable que vous puissiez facilement comprendre le code suivant :

```C
#include <stdio.h>

int main() {
  int x = 10;
  int y = 100;
  printf("x + y = %d", x + y);
  return 0;
}
```

Mais, êtes-vous capables d'expliquer comment ce code fonctionne-t-il à bas niveau ? Et oui, c'est déjà plus difficile pour certains. Nous pouvons rédiger des programmes avec des langages haut niveau comme Haskell, Erlang, Go etc. et être incapable d'en expliquer le fonctionnement après compilation. Avançons-nous donc plus en profondeur, vers le le langage assembleur.

## Préparation

Avant de commencer, nous devons au préalable éclaircir plusieurs points. J'utilise Ubuntu (14.04.1 LTS 64 Bits). De cette manière, mes explications se baseront sur ce système d'exploitation et son architecture. Un processeur implique un ensemble unique d'instructions. J'utilise ici Intel Core i7 870.
Nous nous servirons ici de NASM, que vous pouvez installer sur votre ordinateur avec :

```
$ sudo apt-get install nasm
```

J'utilise ici la version 2.10.09 compilée le 29 décembre 2013.

Pour finir, vous aurez besoin d'un éditeur de texte. Je me sers personnellement d'Emacs avec nasm-mode.el, mais vous êtes libres de choisir celui qui vous correspond le mieux. Si vous souhaitez vous servir d'Emacs, vous pouvez télécharger nasm-mode.el et configurer Emacs de cette manière :

```elisp
(load "~/.emacs.d/lisp/nasm.el")
(require 'nasm-mode)
(add-to-list 'auto-mode-alist '("\\.\\(asm\\|s\\)$" . nasm-mode))
```

C'est tout ce dont nous avons besoin pour le moment.

## Syntaxe d'assembleur NASM

Ici, nous ne mentionnerons que les deux principales sections d'un programme NASM :

* section data
* section text

La section data sert à déclarer des constantes. Ces données ne changent pas au moment de l'exécution de programme. La syntaxe est la suivante :

```assembly
    section .data
```

La section text est pour le code. Elle doit impérativement commencer par global _start, qui indique au kernel où l'exécution du programme démarre.

```assembly
    section .text
    global _start
    _start:
```

Les commentaires sont précédés d'un point-virgule ";". Tous les codes sources NASM contiennent la combinaison des 4 champs suivants :

```
[label:] instruction [opérateur] [; commentaire]
```

Les champs entre crochets sont facultatifs. Une instruction basique NASM est composée de deux parties. La première est le nom de l'instruction qui doit être exécutée, la deuxième est l'opérateur de cette commande. Par exemple :

```assembly
    MOV COUNT, 48 ; Attribuer la valeur 48 à la variable COUNT
```

## Dis bonjour !

Écrivons notre premier programme avec l'assembleur NASM, le traditionnel "Hello World" chez les Anglois.

```assembly
section .data
    msg db      "hello, world!"

section .text
    global _start
_start:
    mov     rax, 1
    mov     rdi, 1
    mov     rsi, msg
    mov     rdx, 13
    syscall
    mov    rax, 60
    mov    rdi, 0
    syscall
```

Pas de panique. Cela ressemble en effet à un printf("Hello World"). Essayons de comprendre comment cela fonctionne. Regardez les deux premières lignes. Nous y avons défini la section data et créé la constante msg que nous pouvons maintenant utiliser dans notre section text que nous déclarons juste après.

La partie la plus intéressante est à partir de la ligne 7. 
Nous connaissons déjà l'instruction mov. Elle prend en compte deux opérateurs et attribue la valeur du second dans la première.
Mais que signifient ces rax, rdi etc. ?
Wikipedia nous dit :

```
Un processeur (ou unité centrale de calcul, UCC ; en anglais central processing unit, CPU) est un composant présent dans de nombreux dispositifs électroniques qui exécute les instructions machine des programmes informatiques.
```

La version anglaise nous explique que le processeur effectue les opérations basiques logiques, d'arithmétiques et d'entrée/sortie du système.
D'accord, mais d'où viennent les données pour ces opérations ? La première réponse est la mémoire. En revanche, récupérer les données de la mémoire ralentit le processeur, puisque cela implique des tâches compliquées pour envoyer les requêtes de données. Pour palier à ce problème, le processeur a ses propres emplacements de mémoire internes que l'on appelle registres. 

![registers](/content/assets/registers.png)

Quand nous écrivons "mov rax, 1", cela signifie que nous attribuons la valeur 1 au registre rax. A présent nous savons ce que sont rax, rdi, rbx etc. Mais comment savoir lesquels utiliser ?

* `rax` - registre temporaire: quand nous effectuons un appel syst-me, rax doit contenir le nombre de l'appel système
* `rdx` - troisième argument de fonctions
* `rdi` - premier argument de fonctions
* `rsi` - pointeur : deuxième argument de fonctions

En d'autres termes, nous faisons juste un appel système `sys_write` : 

```C
size_t sys_write(unsigned int fd, const char * buf, size_t count);
```

On y voit trois arguments :

* `fd` - file descriptor. Il peut prendre pour valeur 0, 1 et 2 pour respectivement l'entrée standard, la sortie standard, et l'erreur standard.
* `buf` - pointe vers un tableau de caractères, qui peut être utilisé pour stocker le contenu obtenu du fichier vers lequel pointe le file descriptor.
* `count` - indique le nombre de bytes qui doivent être écrits du fichier vers le tableau de caractères.

Donc nous savons que l'appel système `sys_write` prend trois arguments. Retrouvons notre programme. Nous attribuons 1 au registre rax, ce qui signifie que nous utiliserons l'appel système `sys_write`. Dans la ligne suivante, nous attribuons 1 au registre rdi, qui sera le premier argument de `sys_write`, 1 - sortie standard. Ensuite nous stockongs le pointeur vers msg dans le registre rsi, qui sera la second argument buf pour `sys_write`. Enfin nous attribuons le dernier (troisème) paramètre (taille de la chaîne de caractères) à rdx. Maintenant nous avons tous les arguments nécessaires et nous pouvons appeler la fonction avec syscall.

Maintenant que nous avons correctement affiché 'Hello World", nous devons quitter le programme. Pour ce faire, nous attribuons 60 au registre rax, et 0 au registre rdi, qui donne le code de sortie. Avec 0, le programme quittera avec succès. 
Pour compiler notre programme, nous prenons notre fichier (ici hello.asm) et nous exécutons les commandes qui suivent :

```
$ nasm -f elf64 -o hello.o hello.asm
$ ld -o hello hello.o
```

En résultera un fichier exécutable "hello" que nous pouvons lancer avec ./hello et qui affiche un merveilleux "Hello World".

