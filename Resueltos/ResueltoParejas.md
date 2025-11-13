# Ejercicio 1

## Datos - Contexto

Ahora las tareas pueden tener parejas y compartir un área de memoria física que ambas mappean con la misma virtual y se va asignando *on-demand* (de manera *lazy*, vaya). Las tareas pueden dejar de ser pareja y cumplen roles diferenciados dentro de una pareja (dentro de su relación).

Si nos guiamos por el diagrama de estados provisto, podemos decir que cada tarea puede encontrarse en uno, y solo uno, de los siguientes estados:

- Sin pareja.
- Buscando pareja.
- En pareja siendo lider.
- En pareja no siendo lider.
- En pareja pero quiere abandonar y es la lider.
- En pareja pero de facto solo porque la no-lider abandonó.

Piden definir *syscalls* para poder crear, aceptar y abandonar parejas, las cuales se comportan de distinta manera dependiendo de los estados de las involucradas.

### Resolución

Vamos con algo general. Defino un nuevo `enum` que representa el estado de pareja de las tareas.

```c
typedef enum {
  TASK_ALONE,
  TASK_LOOKING_FOR_COUPLE,
  TASK_LEAD_IN_COUPLE,
  TASK_NOT_LEAD_IN_COUPLE,
  TASK_LEAD_WANTS_TO_BREAK,
  TASK_LEAD_IN_COUPLE_ALONE
} task_couple_state_t;
```

Donde cada valor que puede tomar el `enum` representa un estado de los que mencioné previamente (los escribí en el mismo orden que antes).

De esta manera, le agrego a las entries de `sched_tasks` un nuevo valor.

Pasarán de ser 

```c
typedef struct {
  int16_t selector;
  task_state_t state;
} sched_entry_t;
```

a

```c
typedef struct {
  int16_t selector;
  task_state_t state;
  task_couple_state_t couple_state;
} sched_entry_t;
```

Pero también me será útil, como veremos después, que las tareas se conozcan entre sí (para manejar la asignación de memoria on-demand).

Así, agrego otra entrada:

```c
typedef struct {
  int16_t selector;
  task_state_t state;
  task_couple_state_t couple_state;
  int couple_id;
} sched_entry_t;
```

El `id` de la tarea que se tiene por pareja, ¿Y si no tiene pareja? ¿Qué va allí? Para mi implementación, me dará igual (me fijaré en el `couple_state` cuando necesite tomar una decisión).

El nuevo valor `couple_state` se inicializará como `TASK_ALONE` para cada tarea. Para el otro en verdad dará igual lo que tenga, pero le doy 0 para que no tenga cualquier basura. Basta modificar la función provista por la cátedra para agregar nuevas tareas de la siguiente manera:

```c
int8_t sched_add_task(uint16_t selector) {
  kassert(selector != 0, "No se puede agregar el selector nulo");

  // Se busca el primer slot libre para agregar la tarea
  for (int8_t i = 0; i < MAX_TASKS; i++) {
    if (sched_tasks[i].state == TASK_SLOT_FREE) {
      sched_tasks[i] = (sched_entry_t) {
        .selector = selector,
	    .state = TASK_PAUSED,
        couple_state = TASK_ALONE //AGREGO ESTA NUEVA LINEA,
        couple_id = 0, //Y ESTA
      };
      return i;
    }
  }
  kassert(false, "No task slots available");
}
```

Es agregar la inicialización como indiqué.

Ahora sí, vamos con las *syscalls*.

Para definir una syscall se necesita definir una entrada en la __IDT__ con su descriptor. Como queremos que puedan ser llamadas por las tareas, pido que el privilegio allí indicado sea `dpl=3`, su resolución se hace siempre a nivel de kernel como veníamos haciendo en el taller. En esta misma línea, basta utilizar la macro `IDT_ENTRY3(n)` para algún `n` válido.

```c
#define IDT_ENTRY3(numero)                                                     \
  idt[numero] = (idt_entry_t) {                                                \
    .offset_31_16 = HIGH_16_BITS(&_isr##numero),                               \
    .offset_15_0 = LOW_16_BITS(&_isr##numero),                                 \
    .segsel = GDT_CODE_0_SEL, /           \
    .type = 0xE,                                                               \
    .dpl = 3,                             \
    .present = 1                                                               \
  }
```

El número indicado puede ser cualquiera que no hayamos utilizado para definir otra interrupción y no sea reservado (`0-32`).

Así, elijo

```c
IDT_ENTRY3(90); //crear_pareja()
IDT_ENTRY3(91); //juntarse_con(id_tarea)
IDT_ENTRY3(92); //abandonar_pareja()
```

y las pongo dentro del cuerpo de la función para inicializar la __IDT__
```c

void idt_init() {
  
  //entries para excepciones y otras cositas del taller quedan como estaban
  ...

  //agrego lo nuevo
  IDT_ENTRY3(90); //crear_pareja()
  IDT_ENTRY3(91); //juntarse_con(id_tarea)
  IDT_ENTRY3(92); //abandonar_pareja()

}
```

Ahora toca implementar las rutinas de resolución para cada una, arranco con `crear_pareja`.

#### `crear_pareja()`

Defino la rutina en `isr.asm`

```

...
extern create_couple
...

global _isr90
_isr90:
    pushad

    call create_couple

    ;obtengo el selector de esta tarea y lo comparo con el que me dio mi funcion de C
    str bx
    cmp ax, bx
    je .fin

    ;si es distinto, cambio a la otra tarea
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]

    .fin:
    popad
    iret
```

Todo el trabajo se lo delego a una función de C auxiliar.

__NOTA:__ `create_couple` NO es la syscall pedida `crear_pareja` y sin querer lo puse en inglés, es una función auxiliar para su definición. La syscall es, estrictamente, `_isr90`.

`create_couple` lo que hará es cambiar el estado de pareja de la tarea actual y retornar el selector de la próxima a ejecutar. Por default va a ser el de la tarea actual, pero podría ser que ella se suspenda y deba pasar a otra según lo indicado por la consigna.

Pueden pasar varias cosas dependiendo del `coupling_state` de la tarea llamadora de la syscall (que se corresponde 1 a 1 con los estados que se mencionan en consigna).

- Si está en pareja, no se hace nada. En mi implementación, sería no hacer nada (y retornar de una) si su estado __NO__ es `TASK_ALONE` ni `LOOKING_FOR_COUPLE` (pues el resto de estados cuentan como estar en pareja, aunque haya cosas medio raras como que una tarea puede estar en pareja pero sola porque la abandonaron, dejo que el grafo provisto me guíe en este caso).
- Si no, se la suspende hasta que alguien le acepte. Esta es la líder de la relación. En mi implementación, sería que la llamadora tiene estado `TASK_ALONE`. 

Notar que jamás una tarea buscando pareja podrá llamar a la syscall, pues estará suspendida. En este orden de ideas, chequear si el estado es `LOOKING_FOR_COUPLE` en el primer se puede omitir. 

Me dicen también que la diferencia entre ser líder o no en una pareja (además de cómo es la evolución entre estados según se llaman a las distintas syscalls), es que la líder puede escribir en la memoria compartida, la otra sólo puede ver.

Así, vamos a C. Escribo en `sched.c`

```c
uint16_t create_couple() {
    task_couple_state_t current_couple_state = sched_tasks[current_task].couple_state; //consigo el estado de pareja actual

    if (current_couple_state != TASK_ALONE) {
        return sched_tasks[current_task].selector; //si la tarea actual no está sola, devuelvo su selector, no paso a ejecutar a otra
    } else {
        sched_tasks[current_task] = TASK_LOOKING_FOR_COUPLE; //si estaba sola, ahora busca pareja 
        sched_disable_task(current_task); //en este caso, se queda pausada hasta que alguien la acepte
        uint16_t next_task_selector = sched_next_task(); //como ya no se estará ejecutando, busco a la siguiente según me diga el scheduller
        return next_task_selector; //y doy su selector para hacer el JMP far y cambiar de tarea
    }
}
```

**NOTA**: remarco que lo único que hay que chequear es si el estado de pareja de la tarea llamadora es `TASK_ALONE` o no, pues jamás será `TASK_LOOKING_FOR_COUPLE`(si tiene ese estado, está pausada!).

#### `juntarse_con(id_tarea)`

Vamos con el assembly

```
_isr91:
    pushad

    push EAX ;pusheo parametro para que use esta funcion de C
    call start_relationship_with
    add ESP, 4 ;poppeamos de la pila
    mov [ESP + EAX_OFFSET], EAX ;actualizamos EAX con lo que escupió la función anterior
    
    popad
    iret
```

Esta syscall recibe un parámetro y retorna algo. Debo establecer una convención para su empleo. 

Como sugiere el código, decido:
- `id_tarea` se pasa por el registro `EAX`
- el valor retornado se devuelve por `EAX` también.

`start_relationship_with` es una función de C de un parámetro. Según la __ABI__ de Linux de 32 bits se pasan de derecha a izquierda por pila.

Yo quiero que la tarea invocadora de la syscall tenga en `EAX` el valor que corresponde (va a coincidir con lo que escupe la función de C). 
No puedo meramente dejarlo sin nada luego del poppeo que hice de la pila, porque el `popad` que le seguiría va a settear los registros de propósito general (incluyendo `EAX`) con lo que tenga guardado. TLDR: me lo pisa.

Así, decido directamente cambiar el valor que tiene la pila guardado para darle a `EAX` y problema resuelto.

Vamos con el core de la función. Pueden pasar varias cosasa, dependiendo del estado de pareja de la tarea invocadora:
- Si ya está en pareja, retorna 1. En mi caso, sería que su estado __NO__ es `TASK_ALONE`.
- Si la otra tarea __NO__ estaba buscando pareja, retorno 1. En mi caso, sería que el estado de *aquella* no es `LOOKING_FOR_COUPLE`.
- Si la otra estaba buscando pareja, se arma la relación con *aquella* como líder.

__NOTA__: nuevamente, para el primer caso estoy obviando qué hacer si la tarea invocadora tuviera estado `LOOKING_FOR_COUPLE`. No es posible, estaría pausada y no podría hacer la syscall $¹$.

Le doy con C.

__PREGUNTAR: PUEDO ASUMIR QUE EL ID PASADO ES VALIDO PARA ALGUNA TAREA?__

```c
uint8_t start_relationship_with(int id_tarea) {

     task_couple_state_t current_couple_state = sched_tasks[current_task].couple_state; //consigo el estado de pareja actual
     task_couple_state_t other_couple_state = sched_tasks[id_tarea].couple_state; //consigo el de la otra

    if (current_couple_state != TASK_ALONE || curreent_couple_state) {
        return 1;
    } else {
        form_couple(current_task, task_id); //que vivan los novios!
        return 0;
    }

}
```

El código es implentación directa de lo que comenté más arriba.

¿Qué debe suceder si se arma la pareja?

- `current_task` pasa al estado `TASK_NOT_LEAD_IN_COUPLE`. Está en pareja pero no es lider
- `task_id` pasa al estado `TASK_LEAD_IN_COUPLE`. Es la líder.
- Se debe garantizar acceso *on-demand* a la memoria compartida a ambas tareas, la líder con permiso de escritura.

¿On-demand? Les asigno memoria física recién cuando quieren entrar a una dir virtual en el rango que me indican: `0xC0C00000` + `4MB` - `1` (el -1 porque la dir virtual de inicio también se cuenta en esos 4 megas).

```c
void form_couple(int not_lead_task_id, int lead_task_id) {
    sched_tasks[lead_task_id].couple_state = TASK_LEAD_IN_COUPLE; //vos sos lead
    sched_tasks[not_lead_task_id].couple_state = TASK_NOT_LEAD_IN_COUPLE; //vos no
    
    //Las hago conocerse entre sí
    sched_tasks[lead_task_id].couple_id = not_lead_task_id;
    sched_tasks[not_lead_task_id].couple_id = lead_task_id;
    
    //habilito a la que estaba esperando pareja
    sched_enable_task(lead_task_id);
}
```

Y con eso bastaría para esta syscall, lo de asignar memoria lo haré modificando la rutina de atención de `page_fault`.

... ahora.

El rango de direcciones virtuales que comparten las tareas es (incluido):

```c
[0xC0C00000, 0xC0C00000 + 2²² - 1] bytes = [0xC0C00000 + 4194304 - 1] bytes
```

Nuestra rutina para atender el `page_fault` tal y como está (de tp) ahora da memoria física a las tareas que quieren acceder al rango virtual ` 0x07000000 - 0x07000FFF`.

Necesito saber sobre la tarea que causó el fault:
- Si está en pareja.
- Si es la líder.

Pues si bien mappearemos la dir virtual a alguna física, "Notar que una vez creada la pareja se debe garantizar que ambas tareas tengan acceso a los 4MB a partir de `0xC0C00000` __a medida que lo requieran__. Sólo la __líder podrá escribir__ y __ambas podrán leer.__"

Redefinamos, pues, el handler de `page_fault`

```c
bool page_fault_handler(vaddr_t virt) {
  print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
  
  //ACA AGREGO LO NUEVO PARA EL PARCIAL

  int this_task_id = ENVIRONMENT->task_id; //obtengo el id de la tarea que causó el page_fault

  task_couple_state_t current_couple_state = sched_tasks[this_task_id].couple_state; //vveo el estado de pareja de esta tarea

  if (0xC0C00000 <= virt && virt <= (0xC0C00000 + 4194304 - 1) {

    if (current_couple_state == TASK_ALONE) {
        //si está sola no puede acceder. 
        return false;
    } else {
        vaddr_t virt_page = virt & 0xFFFFF000; //obtengo la base de la pagina en la que se encuentra esta dir virtual, es quedarme con los bytes mas altos pq cada uno mide 4KB.
        map_shared_couple_page_for(virt_page, this_task_id); //le mapeo la paginita a ella y su pareja
        return true;
    } 

  } else {
    return false;
  }


  //DE ACA PARA ABAJO ES DEL TP, IGNORAR

  //si está en el rango de on-demand, hay que mappearla a la dir fisica indicada
  if (0x07000000 <= virt && virt <= 0x07000FFF) { //es una paginita!!!!
    //conseguioms el cr3 de la tarea actual para saber el page directory
    uint32_t cr3 = rcr3();
    //la dir virtual base de 0x07000000 se mappea a la fisica 0x3000000
    mmu_map_page(cr3, 0x07000000, 0x3000000, MMU_P | MMU_W | MMU_U); //con los privilegios de consigna
    //dev true
    return true;
  } else { //si no esta en rango valido, chau
    return false;
  }

}
```

Falta definir el comportamiento de `map_shared_couple_page_for(virt_page, this_task_id)`.

TLDR: dada esta tarea (que está en pareja), le mappeo a ella y a su pareja (a menos que la hayan dejado) la página virtual dada a una física con los permisos que correspondan según sea lead o no.

Puede ser que esté:
- En pareja (no líder), por lo que a ella le doy sólo lectura.
- En pareja (líder) le doy a ella escritura y lectura, a la otra solo lectura
- En pareja pero sola (líder). En  ese ccaso, a la otra no le mappeo nada.

__Nota:__ También está el caso de "en pareja pero es la líder y quiere bajarse". En ese caso estaría pausada, no puede haber ocasionado el page fault.

```c
void map_shared_couple_page_for(vaddr_t page_vaddr, int task_id) {

    //obtengo el estado de la tarea que me pasaron
    task_couple_state_t given_task_couple_state = sched_tasks[task_id].couple_state;
    
    //separo en los casos que mencioné

    //ésta es la líder
    if (given_task_couple_state == TASK_LEAD_IN_COUPLE) {
        //asigno roles
        int lead_task_id = task_id;
        int not_lead_task_id = sched_tasks[task_id].couple_id;
        //consigo CR3 para hacer el map
        uint32_t lead_cr3 = get_cr3_of(lead_task_id);
        uint32_t not_lead_cr3 = get_cr3_of(not_lead_task_id);
        //consigo una página física disponible para usuarios (son tareas) con la fun del tp
        paddr_t page_paddr = mmu_next_free_user_page(); 
        //mappeo con la función del tp para ello
        mmu_map_page(lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P | MMU_W); //la lead puede escribir
        mmu_map_page(not_lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P); //la otra no :(
    }

    //ésta no es la líder
    if (given_task_couple_state == TASK_NOT_LEAD_IN_COUPLE) {
        //asigno roles
        int not_lead_task_id = task_id;
        int lead_task_id = sched_tasks[task_id].couple_id;
        //consigo CR3 para hacer el map
        uint32_t lead_cr3 = get_cr3_of(lead_task_id);
        uint32_t not_lead_cr3 = get_cr3_of(not_lead_task_id);
        //consigo una página física disponible para usuarios (son tareas) con la fun del tp
        paddr_t page_paddr = mmu_next_free_user_page(); 
        //mappeo con la función del tp para ello
        mmu_map_page(lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P | MMU_W); //la lead puede escribir
        mmu_map_page(not_lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P); //la otra no :(
    }


    //ésta está en una pareja solitaa (ess lider)
    if (given_task_couple_state == TASK_LEAD_IN_COUPLE_ALONE) {
        //asigno roles
        int lead_task_id = task_id;
        //consigo CR3 para hacer el map
        uint32_t lead_cr3 = get_cr3_of(lead_task_id);
        //consigo una página física disponible para usuarios (son tareas) con la fun del tp
        paddr_t page_paddr = mmu_next_free_user_page(); 
        //mappeo con la función del tp para ello
        mmu_map_page(lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P | MMU_W); //la lead puede escribir
    }

    //en cualquier paso, me piden inicializar en 0 toda esa memoria.
    zero_page(page_paddr);
}
```

__NOTA:__ `zero_page` espera una dirección __física__, que es lo que le estoy pasando. Podría haber hecho otro `zero_page_vaddr` que ponga ceros en una página dada su dir virtual pero es mucho lío (habría que pasar también un CR3), aunque sale parecido al copy page (obtengo la física, la mappeo a una virtual temporal, lo pongo en 0, desmappeo esa virtual temporal), me di cuenta que no hace falta. Ya tengo la dir física a llenar de ceros.

Y bueno, falta ver cómo cómo consigo el CR3.

Ya lo vimos en la clase pre-parcial.

```c
uint32_t get_cr3_of(int task_id) {

    tss_t* tss = get_tss_of(task_id);
    return tss->cr3;
}
```

```c
tss_t* get_tss_of(int task_id) {
    uint16_t selector = sched_tasks[task_id].selector;
    uint16_t index = selector >> 3;

    return gdt[index].base;
}
```

Nada: El `CR3` de una tarea se encuentra en su __TSS__, ésta se encuentra en la __GDT__ en algún índice, el índice que corresponde son los bits más altos del selector asociado a esta tarea, que se guarda por tp en `sched_tasks`.

Ahora, vamos con el `zero_page_vaddr`.


$¹$ Invariante de representación intensifies: gracias por todo, Algo 2.

Ahora con la últimas syscall

#### abandonar_pareja()

El assembly

```c
global _isr92
_isr92:
    pushad

    call abandon_couple()

    ;obtengo el selector de esta tarea y lo comparo con el que me dio mi funcion de C
    str bx
    cmp ax, bx
    je .fin

    ;si es distinto, cambio a la otra tarea
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]

    .fin:
    popad
    iret

    popad
    iret
```

Y nuevamente, hay que separar en casos dependiendo del estado del llamador:

-  Si no está en pareja (o sea, tiene `TASK_ALONE`), no pasa nada.
- Si está en pareja y es la lead (`TASK_LEAD_IN_COUPLE`) se queda esperando a ver si la otra también quiere. Se pausa.
- Si está en pareja y no es la lead (`TASK_NOT_LEAD_IN_COUPLE`), se sale nomás.
- Si está en pareja y es la lead pero de facto está solita (`TASK_LEAD_IN_COUPLE_ALONE`), se sale tranqui.

En cualquier caso, las tareas *recién* pierden acceso a la memoria cuando abandonan. Puede que abandonen de golpe o puede que tengan que esperar (el caso de la lead que quiere romper).

Como en un caso debo cambiar de tareas (cuando la lead se bloquea), haré que mi función en C retorne el selector de la siguiente tarea a ejecutar, como hice en `_isr90`. De ahí, hago el JMP para cambiar si es necesario.

```c
uint16_t abandon_couple() {
    //obtengo el estado de esta tarea
    task_couple_state_t current_task_couple_state = sched_tasks[current_task].couple_state;

    //separo en casos

    //si está sola, sigue corriendo
    if (current_task_couple_state == TASK_ALONE) {
        return sched_tasks[current_task].selector;
    }

    //si está con alguien y es lead, se pausa hasta que la otra quiera abandonar
    if (current_task_couple_state == TASK_LEAD_IN_COUPLE) {
        sched_tasks[current_task].couple_state = TASK_LEAD_WANTS_TO_BREAK; //avisa que se quiere ir ;(
        sched_disable_task(current_task); //se pausa
        uint16_t next_task_selector = sched_next_task;
        return next_task_selector; //sigue otra
    }

    //si no es lead y está en pareja, se sale chill, pero pierde acceso a la memoria compartida y cambia el estado de la otra
    if (current_task_couple_state == TASK_NOT_LEAD_IN_COUPLE) {
        int lead_task = sched_tasks[current_task].couple_id; //busco a la pareja
        sched_tasks[current_task].couple_state = TASK_ALONE; //esta pasa a estar solita

        //la otra podia o no estar bloqueada
        task_couple_state_t lead_task_couple_state = sched_tasks[lead_task].couple_state;
        
        //si la lead estaba bien con la relación, pasa a estar solita pero en pareja (?)
        if (lead_task_couple_state == TASK_LEAD_IN_COUPLE) {
            sched_tasks[lead_task].couple_state = TASK_LEAD_IN_COUPLE_ALONE;
        }

        //si la lead quería romper, lo logra y vuelve a correr. Pierde acceso a la memoria de parejas
        if (lead_task_couple_state == TASK_LEAD_WANTS_TO_BREAK) {
            sched_tasks[lead_task].couple_state = TASK_ALONE; //pasa a estar sola
            sched_enable_task(lead_task); //vuelve a la vida
            losses_couple_shared_memory(lead_task); //pierde el acceso a esa memoriaa
        }

        losses_couple_shared_memory(current_task); //la current pierde por default en este caso 

        return sched_task[current_task].selector; //en cualquier caso, sigue corriendo ésta
    }

    //si era una lead en una relación sola, se sale nomás y pierde acceso a la memoria
    if (current_task_couple_state == TASK_LEAD_IN_COUPLE_ALONE) {
        
        sched_tasks[current_task].couple_state = TASK_ALONE;

        looses_couple_shared_memory(current_task);

        return sched_task[current_task].selector; //sigue corriendo ella
    }

}
```

Y cubro todos los casos.

Basta codear ahora cómo es que hacemos para que una tarea pierda el accesos a la memoria de parejas.

Me basta tener el `id` de una página, con eso puedo recuperar su `CR3` y sé qué rango de memoria virtual debo desmappear. Estrictamente, *no sé* cuánta memoria utilizaron las tareas hasta ahora (problema para cuando haga el segundo ejercicio), pero de todas formas, ya sé cuál es el rango al que tienen accesso: `4MB` desde `0xC0C00000`. Algunas de las páginas de ahí puede que no estén mappeadas, pero me da igual, desmappeo a todas. Si justo una de las que intenté desmappear no estaba mappeada, no hago nada y paso a la siguiente.

```c
void looses_couple_shared_memory(int task_id) {
    
    uint32_t cr3 = get_cr3_of(task_id); //ya lo codeé antes

    //veo los 4MB que pueden tener y listo
    for (int i = 0; i < 1024; i++) {
        vaddr_t page_to_unmap = 0xC0C00000 + i*PAGE_SIZE; //consigo una paginita a desmappear
        mmu_unmap_page(cr3, page_to_unmap); //uso la función que hicimos en el tp
    }

}
```

__NOTA:__ `mmu_unmap_page` del tp retorna la dir física que estaba asociada ala dir virtual que le pasaron según la estructura de paginación que define el cr3 dado (si había). Si el page table o page entry por el que se tenía que pasar para llegar a la dir física no estaba `present`, retorna 0 y chau. No estaba desmappeada y listo. Es decir, "no tengo que tener cuidado" con el caso en que justo una `page_to_unmap` en primer lugar no estaba mappeada.

#### Observaciones generales

# Ejercicio 2

## Datos - Contexto

Quiero saber la cantidad de memoria que usa el sistema de parejas con una función que debe correr a nivel de kernel.

```c
uint32_t uso_de_memoria_de_las_parejas();
```

Puedo pensarlo como una syscall (preguntado a profes). Una syscall de nivel 0.

Como antes, entonces le defino una entry en la __IDT__ que esté libre, por ejemplo la 93 para seguir con la numeración de las del ejercicio 1

```c
IDT_ENTRY0(92);
```

Defino su rutina de atención

```c
global _isr92
_isr92:
    pushad

    call uso_de_memoria_de_las_parejas

    mov [ESP + EAX_OFFSET], EAX; muevo al valor que tiene la pila reservado para EAX lo que escupió la función

    popad
    iret
```

Nuevamente, hay condiciones para calcular cuánta es esta memoria.

- si se hace un `abandonar_pareja` y queda una pareja sin participantes, cuenta como si no estuvieran usando recursos.
- sólo se usa la memoria a la que se intentó acceder (la mappeada, pues).
- si dos tareas que están en pareja tienen accesso a `N` bytes de memoria, no lo contabilizamos dos veces (uno por cada tarea), si no una sola.


**IMPORTANTE:** Pregunté y me dijeron que debe devolver la cantidad de memoria utilizada en total por __TODAS__ las tareas. Vamos, la suma sin contar repetidos (por las que están en pareja posta).

Puedo agregar a los entries de `sched_tasks` otro valor que indique la cantidad de memoria de pareja que está utilizando una tarea.

Luego, con el `couple_state`, sé, al momento de hacer la suma, cómo manejar ese "no debo contar dos veces".

La modificación sería:

```c
typedef struct {
  int16_t selector;
  task_state_t state;
  task_couple_state_t couple_state;
  int couple_id;
  uint32_t shared_couple_memory_size;
} sched_entry_t;
```

donde este nuevo atributo se va a inicializar en 0 cuando se cree una tarea.

```c
int8_t sched_add_task(uint16_t selector) {
  kassert(selector != 0, "No se puede agregar el selector nulo");

  // Se busca el primer slot libre para agregar la tarea
  for (int8_t i = 0; i < MAX_TASKS; i++) {
    if (sched_tasks[i].state == TASK_SLOT_FREE) {
      sched_tasks[i] = (sched_entry_t) {
        .selector = selector,
	    .state = TASK_PAUSED,
        couple_state = TASK_ALONE //LO PUSE ANTES,
        couple_id = 0, //ESTO TAMBIEN,
        shared_couple_memory_size = 0; //ESTO ES NUEVO
      };
      return i;
    }
  }
  kassert(false, "No task slots available");
}
```

Noto algo importante: Sólo debo contar una vez la memoria por pareja, puedo entonces plantear que la única que conoce ese valor real es la *lead*. O sea, no voy a tener en cuenta, en una parejita, lo que diga que tiene de memoria la tarea que no es lead.

Así, evito contar dos veces.

Luego, es fácil hacer la función

```c
uint32_t uso_de_memoria_de_las_parejas() {
    uint32_t res = 0;

    for (int i = 0; i < MAX_TASKS; i++) {
        
        //obtengo su estado de pareja
        task_couple_state_t ith_task_couple_state = sched_tasks[i].couple_state;
        
        //solo suma si es lead!
        if (ith_task_couple_state == TASK_LEAD_IN_COUPLE || ith_task_couple_state == TASK_LEAD_IN_COUPLE_ALONE || ith_task_couple_state == TASK_LEAD_WANTS_TO_BREAK) {
            uint32_t ith_shared_couple_memory_size = sched_task[i].shared_couple_memory_size;
            res = res + ith_shared_couple_memory_size;
        }

    }

    return res;
}

```

La cosa es que una *pareja* existe mientras la lead esté dentro de ella. Por eso basta hacer que la lead guarde la memoria. 
- Si la no lead se sale, ella ya no tiene la memoria mappeada, pero la otra sí.
- Si la lead quiere irse, tiene que esperar a que la otra quiere. No se desmappeó su memoria.
- Si ambas están bien juntas, sólo estoy contando a una, no repito.

Pero hay que ver que en todo momento ese `shared_couple_memory_size` es correcto. Ya mostré que se inicializa en 0. Vamos con su actualización.

¿Cuándo se modifica la cantidad de memoria usada? Cuando la mappeo y desmappeo.

Cuando la mappeo, lo hago de a una paginita.

Cuando desmappeo, vuelo todo del tirón.

Así, modifico mi `map_shared_couple_memory` (es lo que usa el `page_fault_handler`) y mi `looses_couple_shared_memory` (es lo que usa el `abandon_couple`).

Para la primer función, voy a hacer que a la líder se le sumen 1024 bytes, el tamaño de una página.

```c
void map_shared_couple_page_for(vaddr_t page_vaddr, int task_id) {

    //obtengo el estado de la tarea que me pasaron
    task_couple_state_t given_task_couple_state = sched_tasks[task_id].couple_state;

    //dir física a la que hay que poner en 0s
    paddr_t page_addr;
    
    //separo en los casos que mencioné

    //ésta es la líder
    if (given_task_couple_state == TASK_LEAD_IN_COUPLE) {
        //asigno roles
        int lead_task_id = task_id;
        int not_lead_task_id = sched_tasks[task_id].couple_id;
        //consigo CR3 para hacer el map
        uint32_t lead_cr3 = get_cr3_of(lead_task_id);
        uint32_t not_lead_cr3 = get_cr3_of(not_lead_task_id);
        //consigo una página física disponible para usuarios (son tareas) con la fun del tp
        page_paddr = mmu_next_free_user_page(); 
        //mappeo con la función del tp para ello
        mmu_map_page(lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P | MMU_W); //la lead puede escribir
        mmu_map_page(not_lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P); //la otra no :(

        sched_tasks[lead_task_id].shared_couple_memory_size += PAGE_SIZE; //NUEVO
    }

    //ésta no es la líder
    if (given_task_couple_state == TASK_NOT_LEAD_IN_COUPLE) {
        //asigno roles
        int not_lead_task_id = task_id;
        int lead_task_id = sched_tasks[task_id].couple_id;
        //consigo CR3 para hacer el map
        uint32_t lead_cr3 = get_cr3_of(lead_task_id);
        uint32_t not_lead_cr3 = get_cr3_of(not_lead_task_id);
        //consigo una página física disponible para usuarios (son tareas) con la fun del tp
        page_paddr = mmu_next_free_user_page(); 
        //mappeo con la función del tp para ello
        mmu_map_page(lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P | MMU_W); //la lead puede escribir
        mmu_map_page(not_lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P); //la otra no :(

        sched_tasks[lead_task_id].shared_couple_memory_size += PAGE_SIZE; //NUEVO
    }


    //ésta está en una pareja solitaa (ess lider)
    if (given_task_couple_state == TASK_LEAD_IN_COUPLE_ALONE) {
        //asigno roles
        int lead_task_id = task_id;
        //consigo CR3 para hacer el map
        uint32_t lead_cr3 = get_cr3_of(lead_task_id);
        //consigo una página física disponible para usuarios (son tareas) con la fun del tp
        page_paddr = mmu_next_free_user_page(); 
        //mappeo con la función del tp para ello
        mmu_map_page(lead_cr3, page_vaddr, page_paddr, MMU_U | MMU_P | MMU_W); //la lead puede escribir

        sched_tasks[lead_task_id].shared_couple_memory_size += PAGE_SIZE; //NUEVO
    }

    //en cualquier paso, me piden inicializar en 0 toda esa memoria.
    zero_page(page_paddr);
}
```

Recuerdo por cómo hice los chequeos en el `page_fault_handler` que la tarea que intentó acceder a esta página virtual está en pareja (y no está pausadaa), por lo que alguno de los ifs da `true` y el `zero_page` no la queda.

El otro momento donde se modifica la cantidad de memoria es cuando se rompe una pareja. En ese caso usé `looses_couple_shared_memory`. Lo actualizo

```c
void looses_couple_shared_memory(int task_id) {
    
    uint32_t cr3 = get_cr3_of(task_id); //ya lo codeé antes

    //veo los 4MB que pueden tener y listo
    for (int i = 0; i < 1024; i++) {
        vaddr_t page_to_unmap = 0xC0C00000 + i*PAGE_SIZE; //consigo una paginita a desmappear
        mmu_unmap_page(cr3, page_to_unmap); //uso la función que hicimos en el tp
    }

    sched_tasks[task_id].shared_couple_memory = 0; //pum, perdió todo

}
```

Si justo la tarea con el id pasado era no lead, el `= 0` en verdad no cambia nada, a esa nunca se le incrementó el contador de size. 

Pero si era la lead que por fin pudo abandonar la relación, se le pone correctamente en 0.

Notar que tareas sin pareja, pues, siempre tienen su size de memoria compartida en 0, y efectivamente, no tienen esa memoria mappeada.