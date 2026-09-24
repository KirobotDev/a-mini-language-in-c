# Implémenter un langage en C

Dans cet article, nous implémentons un petit langage de programmation en C. Le
langage, que j'ai étendu à partir de
[**tinyc**](http://www.iro.umontreal.ca/~felipe/IFT2030-Automne2002/Complements/tinyc.c)
de **Marc Feeley**, est un langage d'instructions. J'ai ajouté une
instruction `print` au langage (ce qui signifie étendre l'analyseur lexical
(tokenizer) et l'analyseur syntaxique (parser), ajouter une nouvelle instruction
machine et modifier l'interpréteur de machine virtuelle en conséquence). J'ai
aussi supprimé la restriction qui n'autorisait que des identifiants
prédéfinis à un seul symbole ; là encore, cela a entraîné des modifications
dans l'analyseur lexical, l'analyseur syntaxique, le compilateur et
l'interpréteur de machine virtuelle. De plus, comme on le verra, j'ai ajouté
au programme un afficheur d'arbre de syntaxe abstraite et un interpréteur
d'arbre de syntaxe abstraite.

Dans ce langage, un programme est simplement une instruction, où une
instruction peut être une instruction conditionnelle, une instruction de boucle,
une instruction d'affichage, une instruction vide, ou zéro ou plusieurs
instructions entre accolades. Chaque instruction est suivie d'un point-virgule.
Pour être précis, la définition est donnée ci-dessous dans la grammaire BNF du
langage. Les non-terminaux sont entre `<>` tandis que les terminaux sont
montrés entre guillemets. Notez que « zéro ou plusieurs » est simplement mis
entre une paire d'accolades (au lieu de l'habituel `*`).

```
 <program>   := <statement>
 <statement> := "if" <paren_expr> <statement>
             := "if" <paren_expr> <statement> "else" <statement>
             := "while" <paren_expr> <statement>
             := "do" <statement> "while" <paren_expr> ";"
             := "print" <paren_expr> ";"
             := "{" { <statement> } "}"
             := <expr> ";"
             := ";"

 <paren_expr> := "(" <expr> ")"
 <expr>       := <test>
              := <id> "=" <expr>
 <test>       := <sum>
              := <sum> "<" <sum>
 <sum>        := <term>
              := <sum> "+" <term>
              := <sum> "-"  <term>
 <term>       := <id>
              := <int>
              := <paren_expr>
 <id>         := <a_finite_sequence_of_acceptable_symbols>
 <num>        := <an_unsigned_decimal_integer>
```

#### L'analyseur lexical (Tokenizer)

Comme on le sait, l'analyseur lexical rend disponible le jeton (token) courant
et permet de passer au jeton suivant. En général, un jeton a un type et une
valeur. Dans notre analyseur lexical, nous avons les types de jetons suivants

```c
enum
{
  DO_SYM,
  ELSE_SYM,
  IF_SYM,
  WHILE_SYM,
  PRINT_SYM,
  LBRA_SYM,
  RBRA_SYM,
  LPAR_SYM,
  RPAR_SYM,
  PLUS_SYM,
  MINUS_SYM,
  LESS_SYM,
  SEMI_SYM,
  EQUAL_SYM,
  NUM_SYM,
  ID_SYM,
  EOI_SYM
};
```

À l'exception de `EOI_SYM`, qui aide pour l'analyse syntaxique ultérieure, tous
les autres types de jetons proviennent de la grammaire. Chaque terminal a un
type de jeton, et de plus les non-terminaux qui génèrent directement des
terminaux (`<id>` et `<int>`) appartiennent aussi à certains types de jetons.
Ces deux derniers ont des valeurs associées que nous conserverons dans un
entier et une chaîne de caractères :

```c
int num_val;
char id_name[100];
```

D'autre part, le jeton courant (plus précisément le type de jeton courant) et
l'obtention du type de jeton suivant sont réalisés par une variable et une
fonction :

```c
int sym;
void next_sym()
{
  // À compléter
}
```

Afin de signaler une erreur de syntaxe, nous aurons la fonction

```c
void syntax_error(char *msg)
{
  fprintf(stderr, "syntax error - %s\n", msg);
  exit(1);
}
```

Écrivons maintenant l'analyseur lexical, c'est-à-dire implémentons
`next_sym`. L'idée de base est d'examiner le caractère courant et de mettre à
jour le type de jeton stocké dans `sym`, puis de passer au caractère suivant.
Dans le cas des nombres et des identifiants, les caractères sont accumulés et
mis respectivement dans `num_val` et `id_name`. Nous aurons donc tout d'abord

```c
int ch = ' ';
void next_ch() { ch = getchar(); }
```

Les espaces sont simplement ignorés, et nous aurons

```c
void next_sym()
{
again:
  switch (ch)
  {
  case ' ':
  case '\n':
    next_ch();
    goto again;
  // À continuer
  }
}
```

Pour continuer, quand `EOF` est atteint, `sym` est mis à jour à `EOI_SYM`.

```c
void next_sym()
{
again:
  switch (ch)
  {
  // ...
  case EOF:
    sym = EOI_SYM;
    break;
  // À continuer
  }
}
```

Dans le cas de `+`, `-`, `=`, etc., `sym` est mis à jour en conséquence, et le
caractère suivant est rendu disponible par un appel à `next_ch()`.

```c
void next_sym()
{
again:
  switch (ch)
  {
  // ...
  case '{':
    next_ch();
    sym = LBRA_SYM;
    break;
  case '}':
    next_ch();
    sym = RBRA_SYM;
    break;
  case '(':
    next_ch();
    sym = LPAR_SYM;
    break;
  case ')':
    next_ch();
    sym = RPAR_SYM;
    break;
  case '+':
    next_ch();
    sym = PLUS_SYM;
    break;
  case '-':
    next_ch();
    sym = MINUS_SYM;
    break;
  case '<':
    next_ch();
    sym = LESS_SYM;
    break;
  case ';':
    next_ch();
    sym = SEMI_SYM;
    break;
  case '=':
    next_ch();
    sym = EQUAL_SYM;
    break;
  // À continuer
  }
}
```

Le cas restant est soit un nombre, soit un identifiant, soit une erreur de
syntaxe. Si le caractère courant est un chiffre, nous accumulons simplement
tous les chiffres suivants, convertissons le jeton en nombre et le stockons
dans `num_val`, et mettons à jour `sym` à `NUM_SYM` :

```c
void next_sym()
{
again:
  switch (ch)
  {
  // ...
  default:
    if (ch >= '0' && ch <= '9')
    {
      num_val = 0;
      while (ch >= '0' && ch <= '9')
      {
        num_val = num_val * 10 + (ch - '0');
        next_ch();
      }
      sym = NUM_SYM;
    }
    // À continuer
  }
}
```

Si par contre le caractère courant est une lettre, nous avons affaire à un
identifiant. Ici, il y a une subtilité : l'identifiant pourrait être l'un des
mots de notre langage, comme `do`, `else`, etc. Ainsi, nous aurons

```c
char *words[] = {"do", "else", "if", "while", "print", NULL};
```

Notez l'ordre dans lequel nous avons mis les mots : l'ordre de `do` correspond
à celui de `DO_SYM`, `else` à `ELSE_SYM`, et ainsi de suite. Il en est ainsi
pour qu'après avoir accumulé les caractères d'un identifiant dans `id_name`
(correctement terminé par `\0`), nous puissions remettre `sym` à `0`, ce qui
correspond à l'indice de `do` dans `words` (et à l'indice de `DO_SYM`). Puis
nous pouvons simplement incrémenter `sym` pour vérifier tous les mots, ce qui
en même temps garantit que `sym` a la bonne valeur si un mot a effectivement
été vu. Sinon, nous avons vu un identifiant et nous mettons simplement `sym`
à `ID_SYM`. Ainsi

```c
void next_sym()
{
again:
  switch (ch)
  {
  // ...
  default:
    // ...
    else if (ch >= 'a' && ch <= 'z')
    {
      int i = 0;
      while ((ch >= 'a' && ch <= 'z') || ch == '_' || (ch >= '0' && ch <= '9'))
      {
        id_name[i++] = ch;
        next_ch();
      }
      id_name[i] = '\0';
      sym = 0;
      while (words[sym] != NULL && strcmp(words[sym], id_name) != 0)
        sym++;
      if (words[sym] == NULL)
        sym = ID_SYM;
    }
    // .. À continuer
  }
}
```

Tout le reste est une erreur de syntaxe dans le langage, et nous avons

```c
void next_sym()
{
again:
  switch (ch)
  {
  // ...
  default:
    // ...
    // ...
    else
      syntax_error("unknown symbol");
  }
}
```

L'analyseur lexical est terminé. La fonction suivante, qui affiche tous les
jetons, devrait être écrite au fur et à mesure que nous étendions l'analyseur
lexical :

```c
void print_tokens()
{
again:
  next_sym();
  switch (sym)
  {
  case DO_SYM:
    printf("DO_SYM \"%s\"\n", id_name);
    goto again;
  case ELSE_SYM:
    printf("ELSE_SYM \"%s\"\n", id_name);
    goto again;
  case IF_SYM:
    printf("IF_SYM \"%s\"\n", id_name);
    goto again;
  case WHILE_SYM:
    printf("WHILE_SYM \"%s\"\n", id_name);
    goto again;
  case PRINT_SYM:
    printf("PRINT_SYM \"%s\"\n", id_name);
    goto again;
  case LBRA_SYM:
    printf("LBRA_SYM\n");
    goto again;
  case RBRA_SYM:
    printf("RBRA_SYM\n");
    goto again;
  case LPAR_SYM:
    printf("LPAR_SYM\n");
    goto again;
  case RPAR_SYM:
    printf("RPAR_SYM\n");
    goto again;
  case PLUS_SYM:
    printf("PLUS_SYM\n");
    goto again;
  case MINUS_SYM:
    printf("MINUS_SYM\n");
    goto again;
  case LESS_SYM:
    printf("LESS_SYM\n");
    goto again;
  case SEMI_SYM:
    printf("SEMI_SYM\n");
    goto again;
  case EQUAL_SYM:
    printf("EQUAL_SYM\n");
    goto again;
  case NUM_SYM:
    printf("NUM_SYM \"%d\"\n", num_val);
    goto again;
  case ID_SYM:
    printf("ID_SYM \"%s\"\n", id_name);
    goto again;
  case EOI_SYM:
    printf("EOI_SYM\n");
    break;
  }
}
```

Voici une simple démonstration avec `{ i=1; while (i<100) i=i+i; }` :

```
LBRA_SYM
ID_SYM "i"
EQUAL_SYM
NUM_SYM "1"
SEMI_SYM
WHILE_SYM "while"
LPAR_SYM
ID_SYM "i"
LESS_SYM
NUM_SYM "100"
RPAR_SYM
ID_SYM "i"
EQUAL_SYM
ID_SYM "i"
PLUS_SYM
ID_SYM "i"
SEMI_SYM
RBRA_SYM
EOI_SYM
```

#### L'analyseur syntaxique (Parser)

Pour écrire l'analyseur syntaxique, nous aurons d'abord besoin de donner des
noms aux constructions sémantiques du langage, ce qui se fait en parcourant la
grammaire et en étiquetant les productions. La liste suivante est complète :

```c
enum
{
  VAR,
  CST,
  ADD,
  SUB,
  LT,
  SET,
  IF,
  IFELSE,
  WHILE,
  DO,
  PRINT,
  EMPTY,
  SEQ,
  EXPR,
  PROG
};
```

Ensuite, nous aurons besoin d'une structure de données pour contenir non
seulement le type d'une construction mais aussi ses composants. Nous aurons
besoin d'au maximum trois données (la construction `IFELSE`) et la structure de
données suivante est adéquate pour toutes les constructions du langage :

```c
typedef struct node
{
  int kind;
  struct node *o1, *o2, *o3;

  union {
    int val;
    char id[100];
  };

} node;
```

Un nouveau `node` est créé en appelant la fonction suivante :

```c
node *new_node(int k)
{
  node *x = malloc(sizeof(node));
  x->kind = k;
  return x;
}
```

À certains moments de l'analyse syntaxique, nous aurons simplement besoin de
consommer un type de jeton attendu, et la fonction suivante est utile :

```c
void consume(int expected)
{
  if (sym == expected)
    next_sym();
  else
    syntax_error("unknown expected");
}
```

L'analyseur syntaxique lui-même est constitué d'un ensemble de fonctions
éventuellement récursives qui s'appellent les unes les autres selon la
grammaire. Pour commencer, `id` crée simplement un nouveau nœud de type `VAR`
et copie le contenu de `id_name` ; il rend aussi disponible le jeton suivant en
appelant `next_sym` :

```c
node *id()
{
  node *x = new_node(VAR);
  strcpy(x->id, id_name);
  next_sym();
  return x;
}
```

Ensuite, `num` est similaire :

```c
node *num()
{
  node *x = new_node(CST);
  x->val = num_val;
  next_sym();
  return x;
}
```

Ensuite, selon sa grammaire, `term` décide quel non-terminal suivre selon le
type de jeton courant. Notez que nous avons besoin d'une déclaration anticipée
de `paren_expr()`.

```c
node *paren_expr();

node *term()
{
  if (sym == ID_SYM)
    return id();
  else if (sym == NUM_SYM)
    return num();
  else
    return paren_expr();
}
```

Vient ensuite `sum`. Selon sa grammaire, il y a trois productions dont aucune
ne commence par un terminal. Cependant, nous pouvons développer une addition ou
une soustraction et voir qu'un terme doit toujours être analysé en premier.
Ensuite, il y a zéro ou plusieurs additions ou soustractions, ce qui se traduit
par une structure de boucle. Ainsi

```c
node *sum()
{
  node *x = term();
  while (sym == PLUS_SYM || sym == MINUS_SYM)
  {
    node *t = x;
    x = new_node(sym == PLUS_SYM ? ADD : SUB);
    next_sym();
    x->o1 = t;
    x->o2 = term();
  }
  return x;
}
```

Faisons une pause ici et essayons de visualiser le travail de l'analyseur
syntaxique. Étant donné une expression comme `1 + 2 - 3`, nous obtiendrons
l'arbre analysé

```
    +
  /   \
 1     -
     /   \
    2     3
```

Le suivant est `test` :

```c
node *test()
{
  node *x = sum();
  if (sym == LESS_SYM)
  {
    node *t = x;
    x = new_node(LT);
    next_sym();
    x->o1 = t;
    x->o2 = sum();
  }
  return x;
}
```

Maintenant, `expr`, qui est un peu délicat. Même si un identifiant semble
pointer vers la deuxième production, si nous développons la première
production, `<test>`, nous voyons qu'elle peut aussi commencer par un
identifiant. Dans ce qui suit, nous vérifierons simplement si le jeton courant
n'est pas un identifiant, auquel cas nous analysons la première production en
renvoyant ce qui sera analysé par `test`. Sinon, nous appelons `test` et
vérifions si nous avons un identifiant suivi du signe égal, auquel cas nous
analysons la clause `SET`, et sinon, c'est aussi ce qui a été analysé par
`test`.

```c
node *expr()
{
  if (sym != ID_SYM)
    return test();

  node *x = test();
  if (x->kind == VAR && sym == EQUAL_SYM)
  {
    node *t = x;
    x = new_node(SET);
    next_sym();
    x->o1 = t;
    x->o2 = expr();
  }
  return x;
}
```

Ensuite, `paren_expr`, c'est probablement la plus simple :

```c
node *paren_expr()
{
  consume(LPAR_SYM);
  node *x = expr();
  consume(RPAR_SYM);

  return x;
}
```

Ensuite vient `statement`, qui a huit productions et est donc la fonction la
plus longue. Sept d'entre elles commencent par un terminal distinct indiquant
quelle production doit être analysée (même s'il y a deux instructions
conditionnelles, on peut distinguer l'une de l'autre grâce au terminal `else`).
La plupart d'entre elles sont simples, à l'exception de la production de
séquence. Elle se traduit par une structure de boucle qui continue à analyser
et à rattacher ce qui a été analysé comme un composant à ce qui va être
analysé. Nous pouvons approximativement voir comment cela fonctionne en
regardant un exemple : la séquence `{ i=1; while (i<100) i=i+i; }` sera
analysée par l'analyseur syntaxique en l'arbre ci-dessous, l'arbre inférieur
étant analysé en premier et collé à l'arbre supérieur comme une branche.

```
         SEQ
       /     \
     SEQ     WHILE
   /    \
EMPTY  EXPR
```

```c
node *statement()
{
  node *x;
  if (sym == IF_SYM)
  {
    next_sym();
    x = new_node(IF);
    x->o1 = paren_expr();
    x->o2 = statement();
    if (sym == ELSE_SYM)
    {
      x->kind = IFELSE;
      next_sym();
      x->o3 = statement();
    }
  }
  else if (sym == WHILE_SYM)
  {
    x = new_node(WHILE);
    next_sym();
    x->o1 = paren_expr();
    x->o2 = statement();
  }
  else if (sym == DO_SYM)
  {
    x = new_node(DO);
    next_sym();
    x->o1 = statement();
    consume(WHILE_SYM);
    x->o2 = paren_expr();
    consume(SEMI_SYM);
  }
  else if (sym == PRINT_SYM)
  {
    x = new_node(PRINT);
    next_sym();
    x->o1 = paren_expr();
    consume(SEMI_SYM);
  }
  else if (sym == LBRA_SYM)
  {
    x = new_node(EMPTY);
    next_sym();
    while (sym != RBRA_SYM)
    {
      node *t = x;
      x = new_node(SEQ);
      x->o1 = t;
      x->o2 = statement();
    }
    next_sym();
  }
  else if (sym == SEMI_SYM)
  {
    x = new_node(EMPTY);
    next_sym();
  }
  else
  {
    x = new_node(EXPR);
    x->o1 = expr();
    consume(SEMI_SYM);
  }

  return x;
}
```

Enfin, `program`. Nous devons nous souvenir de consommer `EOF_SYM`.

```c
node *program()
{
  node *x = new_node(PROG);
  x->o1 = statement();
  consume(EOI_SYM);
  return x;
}
```

Et une fonction qui lance l'analyse syntaxique :

```c
node *parse()
{
  next_sym();
  node *x = program();
  return x;
}
```

La fonction suivante affiche l'arbre de syntaxe abstraite pour nous donner une
idée approximative de son apparence.

```c
void print_ast(node *x)
{
  switch (x->kind)
  {
  case VAR:
    printf("VAR \"%s\" ", x->id);
    break;
  case CST:
    printf("CST \"%d\" ", x->val);
    break;
  case ADD:
    print_ast(x->o1);
    printf("ADD ");
    print_ast(x->o2);
    break;
  case SUB:
    print_ast(x->o1);
    printf("SUB ");
    print_ast(x->o2);
    break;
  case LT:
    print_ast(x->o1);
    printf("LT ");
    print_ast(x->o2);
    break;
  case SET:
    printf("SET ");
    print_ast(x->o1);
    print_ast(x->o2);
    break;
  case IF:
    printf("IF ");
    print_ast(x->o1);
    print_ast(x->o2);
    break;
  case IFELSE:
    printf("IF ");
    print_ast(x->o1);
    print_ast(x->o2);
    printf("ELSE ");
    print_ast(x->o3);
    break;
  case EXPR:
    printf("EXPR ");
    print_ast(x->o1);
    break;
  case SEQ:
    printf("SEQ ");
    print_ast(x->o1);
    print_ast(x->o2);
    break;
  case PRINT:
    printf("PRINT ");
    print_ast(x->o1);
    break;
  case WHILE:
    printf("WHILE ");
    print_ast(x->o1);
    print_ast(x->o2);
    break;
  case DO:
    printf("DO ");
    print_ast(x->o1);
    printf("WHILE ");
    print_ast(x->o2);
    break;
  case PROG:
    printf("PROG ");
    print_ast(x->o1);
    break;
  case EMPTY:
    printf("EMPTY ");
    break;
  default:
    syntax_error("unknown node");
    break;
  }
}
```

À titre d'exemple, nous pouvons voir la sortie de
`{ i=1; while (i<100) i=i+i; }` :

```
PROG SEQ SEQ EMPTY EXPR SET VAR "i" CST "1" WHILE VAR "i" LT CST "100"
EXPR SET VAR "i" VAR "i" ADD VAR "i"
```

Ici, nous voyons le nœud principal `PROG` qui contient une séquence. Comme nous
avons discuté de la façon dont une séquence était analysée, c'est-à-dire
approximativement `{ i=1; while (i<100) i=i+i; }` sera analysé en

```
         SEQ
       /     \
     SEQ     WHILE
   /    \
EMPTY  EXPR
```

ce qui se reflétait dans la sortie.

#### L'interpréteur

Ce langage est un très petit sous-ensemble du langage C et la sémantique de ses
instructions est claire. Le seul point qui doit être discuté est la portée
(scoping). Habituellement, les accolades ouvrent une nouvelle portée et
l'interprétation devrait tenir compte correctement de la portée. Dans le
langage actuel, cependant, nous traiterons simplement les accolades comme un
moyen de grouper des instructions, et en conséquence il n'y a qu'une seule
portée globale, au lieu de différentes portées locales pour chaque paire
d'accolades correspondantes. À titre d'exemple, dans notre langage,
l'instruction `{a = 3; {a = a + a;}; a = a + a; print(a)}` affichera `12` au
lieu de `6` ; la paire interne d'accolades met bien à jour la variable `a`.

Ainsi, au lieu de créer de nouveaux environnements locaux en entrant dans les
accolades, nous aurons un environnement global qui garde les identifiants et
leurs valeurs. L'environnement sera une liste

```c
typedef struct list
{
  char *id;
  int value;
  struct list *next;
} list;

list *env;
```

Nous voulons pouvoir obtenir un identifiant, qui est un élément de la liste, et
aussi pouvoir chercher la valeur d'un identifiant donné :

```c
list *get_id(char *id)
{
  for (list *lst = env; lst; lst = lst->next)
    if (strcmp(lst->id, id) == 0)
      return lst;

  return (list *)NULL;
}

void lookup_error(char *id)
{
  fprintf(stderr, "error looking up %s\n", id);
  exit(1);
}

int lookup_value(char *id)
{
  list *pid = get_id(id);
  if (pid)
    return pid->value;

  lookup_error(id);
  return -1;
}
```

Enfin, nous voulons pouvoir ajouter un identifiant et sa valeur à
l'environnement global. À cause de notre règle de portée, si l'identifiant
existe déjà, sa valeur sera simplement remplacée par la nouvelle valeur ;
sinon, la nouvelle paire nom-valeur sera ajoutée au début de la liste.

```c
void add_id(char *id, int value)
{
  list *pid = get_id(id);
  if (pid)
  {
    pid->value = value;
    return;
  }

  list *lst = malloc(sizeof(list));
  lst->id = id;
  lst->value = value;
  lst->next = env;
  env = lst;
}
```

Maintenant, l'interpréteur. Selon notre grammaire, il devrait y avoir trois
fonctions, `eval_program`, `eval_statement`, et `eval_expr`. D'abord,
`eval_expr` :

```c
int eval_expr(node *x)
{
  node *var;
  int val;

  switch (x->kind)
  {
  case VAR:
    return lookup_value(x->id);
  case CST:
    return x->val;
  case ADD:
    return eval_expr(x->o1) + eval_expr(x->o2);
  case SUB:
    return eval_expr(x->o1) - eval_expr(x->o2);
  case LT:
    return eval_expr(x->o1) < eval_expr(x->o2);
  case SET:
    var = x->o1;
    val = eval_expr(x->o2);
    add_id(var->id, val);
    return val;
  default:
    eval_error();
    return -1;
  }
}
```

Ensuite, `eval_statement` :

```c
void eval_statement(node *x)
{
  switch (x->kind)
  {
  case PRINT:
    printf("%d\n", eval_expr(x->o1));
    break;
  case IF:
    if (eval_expr(x->o1))
      eval_statement(x->o2);
    break;
  case IFELSE:
    if (eval_expr(x->o1))
      eval_statement(x->o2);
    else
      eval_statement(x->o3);
    break;
  case WHILE:
    while (eval_expr(x->o1))
      eval_statement(x->o2);
    break;
  case DO:
    do
      eval_statement(x->o1);
    while (eval_expr(x->o2));
    break;
  case SEQ:
    eval_statement(x->o1);
    eval_statement(x->o2);
    break;
  case EXPR:
    eval_expr(x->o1);
    break;
  case EMPTY:
    break;
  default:
    eval_error();
  }
}
```

Enfin, `eval_program` :

```c
void eval_program(node *x)
{
  switch (x->kind)
  {
  case PROG:
    eval_statement(x->o1);
    break;
  default:
    eval_error();
  }
}
```

Exécutons l'interpréteur sur le programme
`{ i=0; j = 10; while (i<100) print(i=i+j); }`

```
11
21
31
41
51
61
71
81
91
101
```

#### Le compilateur

Un compilateur traduit un programme de notre langage en un ensemble
d'instructions en bytecode qui seront ensuite interprétées. Chaque instruction
est associée à un nombre, et l'ensemble d'instructions qui sera utilisé est

```c
enum
{
  IFETCH,
  ISTORE,
  IPUSH,
  IPOP,
  IADD,
  ISUB,
  ILT,
  IJZ,
  IJNZ,
  IJMP,
  IPRINT,
  IHALT
};
```

L'interprétation du bytecode sera effectuée par une machine virtuelle à pile,
à partir de la première instruction, avec une pile qui contient les valeurs de
calcul. En outre, il y aura deux tableaux supplémentaires pour garder les
variables et leurs valeurs associées. Chaque variable sera placée à un
emplacement spécifique sur un tableau, et la valeur associée à la variable sera
placée à l'emplacement correspondant sur l'autre tableau. Le numéro
d'emplacement sera utilisé comme une instruction de bytecode.

Veuillez noter que nous opérons à un très bas niveau.

Pour commencer, les instructions sont placées dans un tableau appelé
`object`. Comme nous allons pousser continuellement les instructions sur le
dessus du tableau, nous avons besoin d'un pointeur, `here`. La fonction `g`
met simplement un bytecode donné sur le dessus du tableau de code et monte
d'un élément.

```c
typedef char code;
code object[1000], *here = object;

void g(code c) { *here++ = c; }
```

Comme nous l'avons dit, il y a deux tableaux contenant les variables et leurs
valeurs associées. Nous nous restreignons à un maximum d'une centaine de noms,
et donc à une centaine de valeurs.

```c
char names[100][100], (*namespt)[100] = names;

int globals[100];
```

Une opération importante sur `names` est l'obtention de l'indice d'une
variable. Si la variable est déjà dans le tableau, son indice sera renvoyé ;
sinon, la variable est placée sur le dessus du tableau et l'indice de
l'élément du dessus est renvoyé :

```c
int get_index(char *name)
{
  int i;
  for (char(*npt)[100] = names; npt < namespt; npt++)
  {
    i = npt - names;
    if (strcmp(name, names[i]) == 0)
      return i;
  }
  i = namespt++ - names;
  strcpy(names[i], name);
  return i;
}
```

Passons à la génération du code. Nous écrirons une fonction appelée `c` qui
prend un arbre de syntaxe abstraite et génère le code correspondant.

```c
void c(node *x)
{

  switch (x->kind)
  {
    // À continuer
  }
}
```

Puisque `PROG` est le nœud conteneur, travaillons d'abord sur celui-ci. Nous
devons générer le bytecode et mettre `IHALT` à la fin. Ainsi,

```c
void c(node *x)
{

  switch (x->kind)
  {
    case PROG:
      c(x->o1);
      g(IHALT);
      break;
    // À continuer
  }
}
```

Maintenant, selon la grammaire du langage, il y a huit types différents de
nœuds qui pourraient être contenus dans un nœud `PROG`. Le plus facile est
`EMPTY` :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // À continuer
    case EMPTY:
      break;
    // ...
  }
}
```

Prenons `EXPR` comme suivant. Dans ce cas, une valeur est censée être calculée
et mise sur la pile de la machine virtuelle, avant d'être utilisée. Ainsi, nous
générerons les instructions suivies de l'instruction `IPOP`.

```c
void c(node *x)
{

  switch (x->kind)
  {
    // À continuer
    case EXPR:
      c(x->o1);
      g(IPOP);
      break;
    // ...
  }
}
```

Il y a plusieurs expressions possibles. Commençons par `VAR`. Étant donné une
variable, comme il y a déjà une valeur associée, nous générons simplement
`IFETCH` et l'indice de l'emplacement de la variable :

```c
void c(node *x)
{

  switch (x->kind)
  {
    case VAR:
      g(IFETCH);
      g(get_index(x->id));
      break;
    // ...
  }
}
```

Ensuite, `CST`. Dans ce cas, nous générons `IPUSH` et la valeur à mettre sur la
pile de la machine virtuelle :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case CST:
      g(IPUSH);
      g(x->val);
      break;
    // À continuer
    // ...
  }
}
```

Ensuite viennent `ADD` et `SUB` :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case ADD:
      c(x->o1);
      c(x->o2);
      g(IADD);
      break;
    case SUB:
      c(x->o1);
      c(x->o2);
      g(ISUB);
      break;
    // À continuer
    // ...
  }
}
```

`LT` est similaire :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case LT:
      c(x->o1);
      c(x->o2);
      g(ILT);
      break;
    // À continuer
    // ...
  }
}
```

Enfin, `SET`. Ici, nous calculons d'abord la valeur à définir, puis nous la
stockons, puis nous générons l'indice de la variable :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case SET:
      c(x->o2);
      g(ISTORE);
      g(get_index(x->o1->id));
      break;
    // À continuer
    // ...
  }
}
```

Revenons maintenant aux nœuds qui pourraient être contenus dans le nœud
`PROG`. Nous avons implémenté `EMPTY` et `EXPR`. Le suivant facile est
`PRINT` :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case PRINT:
      c(x->o1);
      g(IPRINT);
      break;
    // À continuer
    // ...
  }
}
```

`SEQ` est aussi simple :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case SEQ:
      c(x->o1);
      c(x->o2);
      break;
    // À continuer
    // ...
  }
}
```

Ensuite, passons à `IF`. La clé pour traduire ce nœud est d'utiliser
l'instruction `IJZ`, qui signifie « sauter si zéro » (jump if zero). Avant
d'écrire le code, nous allons regarder un exemple, `if (1 < 10) print(10);`.
Ici, la condition `1 < 10` sera traduite en une série d'instructions, que nous
marquons `X` ci-dessous :

```
... |X| | |...
```

`print(10)` doit aussi être traduit, mais que la séquence d'instructions doive
ou non être exécutée dépend du fait que la condition est vraie (`1`), autrement
dit la séquence d'instructions doit être sautée si la condition est fausse
(`0`). Pour obtenir cet effet, nous mettons `IJZ` suivi d'un trou, et nous
générons les instructions pour `print(10)`. Le trou est destiné à contenir le
nombre d'instructions à sauter. Nous pouvons garder l'illustration suivante à
l'esprit :

```
... |X|IJZ| |X|...
```

Pour créer un trou, nous avons la fonction

```c
code *hole() { return here++; }
```

Et pour calculer le nombre d'étapes :

```c
void fix(code *src, code *dst) { *src = dst - src; }
```

Le code pour traduire `IF` est :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case IF:
      c(x->o1);
      g(IJZ);
      p1 = hole();
      c(x->o2);
      fix(p1, here);
      break;
    // À continuer
    // ...
  }
}
```

Ensuite, nous traduirons `IFELSE`. Nous utiliserons un exemple concret,
`if (1<10) print(1); else print(10);`, pour suivre. Le code pour `1<10`,
`print(1)` et `print(10)` devra être traduit. Dans une instruction générale,
une des conséquences sera sautée selon la condition. Ainsi, nous placerons une
instruction de saut et un trou devant chacune. Mais le deuxième saut est un
saut simple. Nous utilisons donc l'instruction `IJMP`.

```
...|X|IJZ| |X|JMP| |X| |...
```

Les adresses à conserver dans les trous doivent être calculées aux bons
endroits :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case IFELSE:
      c(x->o1);
      g(IJZ);
      p1 = hole();
      c(x->o2);
      g(IJMP);
      p2 = hole();
      fix(p1, here);
      c(x->o3);
      fix(p2, here);
      break;
    // À continuer
    // ...
  }
}
```

Maintenant, traduisons `WHILE`. Cela fonctionne presque comme une condition.
Nous aurons

```
|X|IJZ| |X|IJMP|...
```

Nous devons être prudents avec le calcul des adresses de saut :

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case WHILE:
      p1 = here;
      c(x->o1);
      g(IJZ);
      p2 = hole();
      c(x->o2);
      g(IJMP);
      fix(hole(), p1);
      fix(p2, here);
      break;
    // À continuer
    // ...
  }
}
```

Enfin, `DOWHILE`. Ici, `JNZ`, ou « sauter si non zéro » (jump if not zero),
est utilisé.

```c
void c(node *x)
{

  switch (x->kind)
  {
    // ...
    case DO:
      p1 = here;
      c(x->o1);
      c(x->o2);
      g(JNZ);
      fix(hole(), p1);
      break;
    // ...
  }
}
```

#### L'interpréteur de machine virtuelle

Les instructions en bytecode générées par le compilateur écrit ci-dessus sont
interprétées par un interpréteur de machine virtuelle à pile. Il y a une pile
pour contenir les valeurs de calcul, et les instructions doivent être
interprétées du début à la fin.

```c
void run()
{
  int stack[1000], *sp = stack;
  code *pc = object;
again:
  switch (*pc++)
  {
    // À continuer ...
  }
}
```

Les instructions suivantes manipulent directement la pile ou le tableau de
valeurs :

```c
void run()
{
  int stack[1000], *sp = stack;
  code *pc = object;
again:
  switch (*pc++)
  {
  case IFETCH:
    *sp++ = globals[*pc++];
    goto again;
  case ISTORE:
    globals[*pc++] = sp[-1];
    goto again;
  case IPUSH:
    *sp++ = *pc++;
    goto again;
  case IPOP:
    --sp;
    goto again;
  case IADD:
    sp[-2] = sp[-2] + sp[-1];
    --sp;
    goto again;
  case ISUB:
    sp[-2] = sp[-2] - sp[-1];
    --sp;
    goto again;
  case ILT:
    sp[-2] = sp[-2] < sp[-1];
    --sp;
    goto again;
  case IPRINT:
    printf("%d\n", *--sp);
    goto again;
    // À continuer ...
  }
}
```

Les instructions restantes impliquent l'utilisation du nombre d'étapes pour
sauter quand c'est nécessaire :

```c
void run()
{
  int stack[1000], *sp = stack;
  code *pc = object;
again:
  switch (*pc++)
  {
    // ...
  case IJZ:
    if (*--sp == 0)
      pc += *pc;
    else
      pc++;
    goto again;
  case IJMP:
    pc += *pc;
    goto again;
  case IJNZ:
    if (*--sp != 0)
      pc += *pc;
    else
      pc++;
    goto again;
  }
}
```

La machine virtuelle est terminée. Étant donné un programme, comme
`{ i = 0; do { i = i + 10; print(i);} while (i < 50);}`, elle produira la
sortie

```
10
20
30
40
50
```
