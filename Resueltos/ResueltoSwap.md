c
/*
En caso de que querramos un valor actual de la tarea en una interrupcion, hay que buscarlo en estos lugares
|            | +                          ┌───> PUSHAD:
| SS_3       | ───┐                       │ | EAX | +0x1C
| ESP_3      |    │                       │ | ECX | +0x18
| EFLAGS     |    ├ Apilado por el        │ | EDX | +0x14 [5]
| CS_3       |    │ cambio de nivel       │ | EBX | +0x10 [4]
| EIP_3      |    │ de privilegio         │ | ESP | +0x0C [3]
| Error Code | ───┘                       │ | EBP | +0x08 [2]
| PUSHAD     | ───────────────────────────┘ | ESI | +0x04 [1]
| ...        |                              | EDI | +0x00 [0]
*/




// SWAP:
// si swap_id es 0 significa que no se llamó a swap
void swap(int8_t task_id_to_swap) {
    // Si la otra tarea no me está esperando, seteo yo la espera
    if(sched_tasks[task_id_to_swap].swap_id != current_task) {
        sched_tasks[current_task].swap_id = task_id_to_swap;
        sched_tasks[current_task].state = TASK_WATING_FOR_SWAP;
        cambiar_tarea();
    }
    // sino me está esperando y hay que cambiar registros
    tss_t* current_tss = obtenerTSS(sched_tasks[current_task].selector);
    tss_t* swap_tss = obtenerTSS(sched_tasks[task_id_to_swap].selector);
    tss_t copy_of_swap = *swap_tss;
    uint32_t* pila = current_tss->esp;

    swap_tss->eax = pila[7];
    swap_tss->ecx = pila[6];
    swap_tss->edx = pila[5];
    swap_tss->ebx = pila[4];
    swap_tss->esi = pila[1];
    swap_tss->edi = pila[0];

    pila[7] = copy_of_swap.eax;
    pila[6] = copy_of_swap.ecx;
    pila[5] = copy_of_swap.edx;
    pila[4] = copy_of_swap.ebx;
    pila[1] = copy_of_swap.esi;
    pila[0] = copy_of_swap.edi;

    sched_tasks[task_id_to_swap].state = TASK_RUNNABLE;

}
tss_t* obtenerTSS(uint16_t selector){
    uint8_t GDT_index = selector >> 3;
    gdt_entry_t TSS_descriptor = gdt[GDT_index];
    tss_t* tss = (tss_t*) (TSS_descriptor.base_15_0 | TSS_descriptor.base_23_16 << 16 | TSS_descriptor.base_31_24 << 24);
    return tss; 
}
