Malloco

1.Detallar todos los cambios que es necesario realizar sobre el kernel para que una tarea de nivel usuario pueda pedir memoria y liberarla, asumiendo como ya implementadas las funciones mencionadas. Para este ejercicio se puede asumir que garbage_collector está implementada y funcionando.

Primero para que una tarea puede pedir y liberar memoria va a necesitar una syscall

**idt.c**

```
IDT_ENTRY3(99);
IDT_ENTRY3(100);
```

La syscall 99, reponsable de reservar memoria, toma como parametro por eax la cantidad de bytes a reservar (4 KB = 4 × 1024 = 4096 bytes)

**isr.asm**
```asm
global _isr99
_isr99:
    pushad
    
    push eax
    call malloco
    add esp, 4

    mov [esp+28], eax 
    
    popad
    iret
```

//El tema del return no me quedaria muy claro

La syscall 100, reponsable de liberar memoria, toma como parametro por eax la cantidad la direccion virtual de memoria a librar

```asm
global _isr100
_isr100:
    pushad

    push eax
    call chau
    add esp, 4

    popad
    iret
```

Lazy allocation, dejamos el isr.asm igual y modificamos el page fault handler

**mmu.c**

```c
bool page_fault_handler(vaddr_t virt)
{
  print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
  // Chequeemos si el acceso fue dentro del area on-demand
  // En caso de que si, mapear la pagina

  if ((uint32_t)ON_DEMAND_MEM_START_VIRTUAL <= virt && virt <= (uint32_t)ON_DEMAND_MEM_END_VIRTUAL)
  {

    uint32_t cr3 = rcr3();

    mmu_map_page(cr3, ON_DEMAND_MEM_START_VIRTUAL, ON_DEMAND_MEM_START_PHYSICAL, MMU_W | MMU_U);
    
    return true;
  }
  
  reservas_por_tarea *reserva_tarea = dameReservas(current_task);  //este me da el size y el array
  reserva_t *reservas = reserva_tarea -> array_reservas;
  for(int i = 0; i < reserva_tarea-> reservas_size; i++){
    
    if(reservas[i].virt <= virt && virt < reservas[i].virt + reservas[i].tamanio){ //Esta dentro de esta reserva
        vaddr_t direccion_truncada = virt & 0xFFFFF000;  //Nos queda multiplo de 4kb y ni haria falta+
        paddr_t addr = mmu_next_free_user_page();
        mmu_map_page(cr3, direccion_truncada, addr, MMU_W | MMU_U);
        zero_page(addr);
        return true;
    }
  }
    //ageregaron la logica de que se pausa y cambia en vez de dar error
  sched_tasks[current_task].state = TASK_PAUSED;

  for(int i = 0; i < reserva_tarea->reservas_size; i++){
        chau(reservas[i].virt); 
    }


  return false;
}
```

**isr.asm**
```nasm
global _isr14

_isr14:
  pushad
  mov eax, cr2
  push eax
  call page_fault_handler
  add esp, 4 ;restaurar stack

  cmp ax, 0
  jnz .handled

  .error:
  call sched_next_task
  mov word [sched_task_selector], ax
  jmp far [sched_task_offset]

  .handled:
	popad
  add esp, 4 ; error code
  iret
```


2.Detallar todos los cambios que es necesario realizar sobre el kernel para incorporar la tarea garbage_collector si queremos que se ejecute una vez cada 100 ticks del reloj. Incluir una posible implementación del código de la tarea.

Para crear una tarea de nivel 0 agregamos
**task.c**
```c
tss_t tss_create_kernel_task(paddr_t code_start) {
    vaddr_t stack = mmu_next_free_kernel_page();
    return (tss_t) {
        .cr3 = create_cr3_for_kernel_task(),
        .esp = stack + PAGE_SIZE,
        .ebp = stack + PAGE_SIZE,
        .eip = (vaddr_t)code_start,
        .cs = GDT_CODE_0_SEL,
        .ds = GDT_DATA_0_SEL,
        .es = GDT_DATA_0_SEL,
        .fs = GDT_DATA_0_SEL,
        .gs = GDT_DATA_0_SEL,
        .ss = GDT_DATA_0_SEL,
        .ss0 = GDT_DATA_0_SEL,
        .esp0 = stack + PAGE_SIZE,
        .eflags = EFLAGS_IF,
    };
}
```
**tasks.c**
```c
uint16_t selector_garbage_collector;

create_garbage_collector() {
    size_t gdt_id;
    for (gdt_id = GDT_TSS_START; gdt_id < GDT_COUNT; gdt_id++) {
        if (gdt[gdt_id].p == 0) break;
    }
    kassert(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");
    //Una vez con la entrada de la gdt, el tss descriptor

    selector_garbage_collector = gdt_id << 3;

    int8_t task_id = sched_add_task(gdt_id << 3);
    tss_tasks[task_id] = tss_create_kernel_task(&garbatss_tge_collector_loop);
    gdt[gdt_id] = tss_gdt_entry_for_task(&tss_tasks[task_id]);
    return task_id;
}
```
Si nunca llamas sched_enable_task queda pausada por defecto, pero yo la voy a forzar a correr, esto evita que el sched la llame por round robin

**mmu.c**
```c
uint32_t create_cr3_for_kernel_task(){
    //inicializar pd
    paddr_t *task_page_dir = mmu_next_free_kernel_page();
    zero_page(task_page_dir);

    //paddr_t cr3 = (page_dir & 0xFFFFF000);  //ESTO HAY QUE AGREGARLO??, no estan limpios ya

    //identity mapping
    for (uint32_t i = 0; i < identity_mapping_end; i += PAGE_SIZE) {
        mmu_map_page(task_page_dir,i, i, MMU_W);
    }
    return task_page_dir;
}
```

Contador y logica para elegir esta tarea

**garbage_collector.c**
```c
void garbage_collector_loop(void) {
	while (true) {
        for(int task_id = 0; task_id < MAX_TASKS; task_id++){
            reservas_por_tarea *reserva_tarea = dameReservas(task_id);
            reserva_t *reservas = reserva_tarea -> array_reservas;
            for(int i = 0; i < reserva_tarea-> reservas_size; i++){
                if(reservas[i].estado == 2){
                    reservas[i].estado = 3;
                    for (uint32_t addr = reservas[i].virt; addr < reservas[i].virt + reservas[i].tamanio; addr += PAGE_SIZE) {
                        mmu_unmap_page(task_selector_to_CR3(sched_task[task_id].selector), addr);
                    }
                }
            }
        }
	}
}
```

**tasks.c**
```c
uint32_t task_selector_to_CR3(uint16_t selector) {
    uint16_t index = selector >> 3; // Sacamos los atributos
    gdt_entry_t* taskDescriptor = &gdt[index]; // Indexamos en la gdt
    tss_t* tss = (tss_t*)((taskDescriptor->base_15_0) |
                            (taskDescriptor->base_23_16 << 16) |
                            (taskDescriptor->base_31_24 << 24));
    return tss->cr3;
}
```


**sched.c**
```c
static uint8_t contador_de_ticks = 0;

void increment_ticks(){
    contador_de_ticks++;
}

void clear_ticks(){
    contador_de_ticks = 0;
}

uint16_t sched_next_task(void) {
    
    //Primero vemos si tenemos que correr el garbage collector
    if(contador_de_ticks >= 100){ //Nunca deberia ser mayor, pero alguna interrupcion podria interferir, mejor asegurar.
        clear_ticks();
        return selector_garbage_collector;
    }
    ...
}

```

**isr.asm**
```asm
_isr32:
    pushad
    call pic_finish1
    
    call next_clock
    
    call increment_ticks();

    call sched_next_task
    ....
```


3.
Ejercicio 3:
a)Indicar dónde podría estár definida la estructura que lleva registro de las reservas (5 puntos)
Yo creo que podria definiste en tasks o idt tal vez.
b)Dar una implementación para malloco (10 puntos)

Hay algo que se llama reservas_global, que este tiene el array de reservas_por_tarea

**idt.c***
```c
%define start_v_dir = 0xa10c0000

void* malloco(size_t size){

    reservas_por_tarea *reserva = &reservas_global[current_task];
    reserva_t* arr = reserva->array_reservas;

    //ver ultima dir
    vaddr_t ultima_dir;
    if(reserva->reservas_size == 0){
        ultima_dir = start_v_dir;
    }else{
        reserva_t ult_reserva = arr[reserva->reservas_size-1];
        ultima_dir = ult_reserva.virt + ult_reserva.tamanio;
    }

    
    uint32_t mem_usada = (ultima_dir + size) - start_v_dir;

    if(mem_usada > 4 * 1024 * 1024){//4mb = 4 * 1024 * 1024
        return NULL;
    } 
    arr[reserva->reservas_size]= {.virt = ultima_dir, .tamanio = size, .estado = 1};
    reserva -> reservas_size += 1;
    return ultima_dir;
}
```

chau


**idt.c***
```c
void chau(vaddr_t virt) {
    if(!esMemoriaReservada(virt)) return;
    
    reservas_por_tarea *reserva = &reservas_global[current_task];    
    reserva_t* array = reserva->array_reservas;
    for(uint32_t i = 0; i < reserva->reservas_size; i++){
        if(array[i].virt == virt && array[i].estado == 1){
            array[i].estado = 2;
            return;
        }
    }
}
```
