# Practica-Instrucciones
# Trabajo Práctico: Testeo de Instrucciones 

En este documento se detallan  casos de prueba diseñados para testear el correcto funcionamiento de las instrucciones de la arquitectura de la CPU RTM32. Se agruparon instrucciones complementarias en cada caso para verificar no solo el funcionamiento aislado, sino también la interacción entre la ALU, los registros y la memoria.

Las instrucciones testeadas son (15 de momento): `LUI` (mediante `ORI/H`), `ORI`, `XOR`, `NOR`, `ADDI`, `SUB`, `SRA`, `SW`, `SB`, `LHU`, `BEQ`, `BGT`, `SLTI`, `J`, `JR`.

Para cada caso se provee la lógica en ensamblador, la traducción exacta a código máquina hexadecimal (respetando la codificación de formatos R, I, L y J) y los comandos de Telnet para inyectar en el debugger virtual de rtm32.

---

# Caso 1: Carga de Constantes y Enmascaramiento Lógico
## Descripción: Qué estoy testeando
Se testea el uso de las instrucciones lógicas y de formato L para cargar un número de 32 bits en dos pasos, aplicarle una máscara usando un OR exclusivo (`XOR`), y luego negar todos sus bits utilizando la operación `NOR` con el registro `$zero`.
## Instrucciones: instrucciones que usé durante el test
`LUI` (simulada vía `ORI/H`), `ORI` (Or Immediate), `XOR` (Exclusive Or), `NOR` (Not Or)

## Precondiciones:
- Reiniciar el estado de la CPU desde el debugger (`reset`).
- Fijar el Program Counter en 0 (`set pc 0x00000000`).
- Setear el registro `$12` ($t4) con una máscara completa de 1s para el XOR: `set r12 0xFFFFFFFF`.
- Inyectar el código máquina hexadecimal en la memoria (iniciando en la dirección `0x00000000`).

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r12 0xFFFFFFFF
set [0x00000000] 0x281500FF
set [0x00000004] 0x2A94AA55
set [0x00000008] 0x0298B00A
set [0x0000000C] 0x02C0D00B
```

## Code
**Ensamblador MIPS/STX4:**
```mips
ORI/H $10, $0, 0x00FF   # Carga 0x00FF en la parte alta de $10 (actúa como LUI). $10 = 0x00FF0000
ORI $10, $10, 0xAA55	# Hace OR con la parte baja (h=0). $10 = 0x00FFAA55
XOR $11, $10, $12   	# $11 = 0x00FFAA55 XOR 0xFFFFFFFF = 0xFF0055AA
NOR $13, $11, $0    	# Niega $11 haciendo NOR con $zero. $13 = 0x00FFAA55
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x281500FF  # ORI/H $10, $0, 0x00FF (Opcode 5, rs=0, rt=10, h=1, imm=0xFF)
0x00000004 : 0x2A94AA55  # ORI $10, $10, 0xAA55  (Opcode 5, rs=10, rt=10, h=0, imm=0xAA55)
0x00000008 : 0x0298B00A  # XOR $11, $10, $12 	(Opcode 0, rs=10, rt=12, rd=11, func=10)
0x0000000C : 0x02C0D00B  # NOR $13, $11, $0  	(Opcode 0, rs=11, rt=0, rd=13, func=11)
```

## Postcondiciones:
- Avanzar 4 ciclos en la CPU usando `step 4`.
- Ejecutar el comando `registers` para volcar el estado del procesador.
- Leer `$10`: debe contener `0x00FFAA55`.
- Leer `$11`: debe contener `0xFF0055AA`.
- Leer `$13`: debe contener `0x00FFAA55`.

## Conclusiones:
Anduvo correctamente. Se observó que el uso del bit `h=1` (bit 16 del formato L) permite operar explícitamente en los 16 bits superiores (MSB) de forma análoga a un Load Upper Immediate (LUI), validando completamente la arquitectura propuesta. La negación mediante `NOR` contra el registro hardwired `$0` funciona perfectamente como un operador NOT lógico.

---

# Caso 2: Aritmética Negativa y Desplazamientos
## Descripción: Qué estoy testeando
Se testea la correcta generación y manipulación de números negativos en complemento a dos, sumas aritméticas con valores inmediatos y desplazamientos aritméticos a la derecha para corroborar que el bit de signo se propaga.
## Instrucciones: instrucciones que usé durante el test
`SUB` (Subtract), `ADDI` (Add Immediate), `SRA` (Shift Right Arithmetic)

## Precondiciones:
- Resetear el simulador y acomodar el PC (`reset` -> `set pc 0`).
- Setear en el debugger el registro `$10` ($t2) con el valor `4`: `set r10 0x00000004`.
- Setear en el debugger el registro `$11` ($t3) con el valor `0` (o usar `$0`).

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000004
set [0x00000000] 0x0014C01D
set [0x00000004] 0x0B1A000A
set [0x00000008] 0x0018E082
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SUB $12, $0, $10    	# $12 = $0 - $10 (0 - 4 = -4 o 0xFFFFFFFC)
ADDI $13, $12, 10   	# $13 = -4 + 10 = 6
SRA $14, $12, 1     	# Shift Right Arithmetic a -4. Debería dar -2 (0xFFFFFFFE)
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0014C01D  # SUB $12, $0, $10 (Opcode 0, rs=0, rt=10, rd=12, func=29)
0x00000004 : 0x0B1A000A  # ADDI $13, $12, 10 (Opcode 1, rs=12, rt=13, imm=10)
0x00000008 : 0x0018E082  # SRA $14, $12, 1   (Opcode 0, rs=0, rt=12, rd=14, aux=1, func=2)
```

## Postcondiciones:
- Avanzar la CPU con `step 3`.
- Chequear con el debugger los registros de destino con el comando `registers`:
- Registro `$12`: debe contener `0xFFFFFFFC` (-4).
- Registro `$13`: debe contener `0x00000006`.
- Registro `$14`: debe contener `0xFFFFFFFE` (-2).

## Conclusiones:
Anduvo. La propagación del bit más significativo (el de signo) en `SRA` se mantiene tal como se espera en un desplazamiento aritmético para números negativos. La ALU procesa sumas algebraicas en complemento a dos de forma transparente.

---

# Caso 3: Acceso asimétrico a Memoria y Endianness
## Descripción: Qué estoy testeando
Se testea la escritura de datos de 32 bits a memoria (Store Word), sobreescritura de un solo byte en la misma base (Store Byte) y posterior lectura asimétrica de media palabra (Load Half Unsigned) para corroborar la alineación y el esquema de almacenamiento de bytes de la CPU.
## Instrucciones: instrucciones que usé durante el test
`SW` (Store Word), `SB` (Store Byte), `LHU` (Load Half Unsigned)

## Precondiciones:
- Setear un puntero base en `$15` apuntando a una dirección válida de RAM (ej. 0x1000): `set r15 0x00001000`.
- Setear el dato original de 32 bits en `$16`: `set r16 0x11223344`.
- Setear el byte a sobreescribir en `$17`: `set r17 0x000000FF`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r15 0x00001000
set r16 0x11223344
set r17 0x000000FF
set [0x00000000] 0x4C1E0000
set [0x00000004] 0x5C5E0000
set [0x00000008] 0x6C9E0000
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SW  $16, $15, 0     	# M[$15] = 0x11223344. Ocupa direcciones 0x1000 a 0x1003.
SB  $17, $15, 0     	# M[$15][7:0] = 0xFF. Modifica la parte menos significativa.
LHU $18, $15, 0     	# Carga los 2 bytes inferiores de la dirección 0x1000 en $18 sin signo.
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x4C1E0000  # SW  $16, $15, 0 (Opcode 9, rs=16, rt=15, imm=0)
0x00000004 : 0x5C5E0000  # SB  $17, $15, 0 (Opcode 11, rs=17, rt=15, imm=0)
0x00000008 : 0x6C9E0000  # LHU $18, $15, 0 (Opcode 13, rs=18, rt=15, imm=0)
```

## Postcondiciones:
- Avanzar la CPU con `step 3`.
- Leer `$18` en el debugger para verificar cómo la CPU ensambla la media palabra.
- Si el micro es Little-Endian, el byte en `0x1000` se modificó a `FF`, por lo que la media palabra (`0x1001:0x1000`) debería leerse como `0x000033FF` en `$18`.
- Inspeccionar el bloque de memoria física usando el comando del debugger: `examine xw 0x1000`.

## Conclusiones:
Anduvo. Los accesos con `SW` respetaron la alineación de palabra (dirección múltiplo de 4). El resultado de la combinación de `SW -> SB -> LHU` nos permitió validar de forma empírica el endianness (ordenamiento de bytes en memoria) implementado en la CPU.

---

# Caso 4: Bifurcaciones y Condiciones
## Descripción: Qué estoy testeando
Se testea el mecanismo de control de flujo usando saltos condicionales respecto al PC, el registro de comparación con inmediatos, un salto incondicional largo (`J`) y un salto indirecto mediante registro (`JR`).
## Instrucciones: instrucciones que usé durante el test
`SLTI` (Set Less Than Immediate), `BEQ` (Branch Equal), `BGT` (Branch Greater Than), `J` (Jump), `JR` (Jump Register)

## Precondiciones:
- Setear `$10` con el valor `50`: `set r10 0x00000032`.
- Setear `$11` con el valor `0`: `set r11 0x00000000`.
- Setear `$20` con una dirección de memoria lejana (ej 0x0400): `set r20 0x00000400`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000032
set r11 0x00000000
set r20 0x00000400
set [0x00000000] 0xB2980064
set [0x00000004] 0x83000002
set [0x00000008] 0x9A960001
set [0x0000000C] 0x0AD60064
set [0x00000010] 0x10000100
set [0x00000400] 0x0500000E
```

## Code
**Ensamblador MIPS/STX4:**
```mips
# Iniciamos en 0x00000000
SLTI $12, $10, 100  	# $10 (50) < 100 -> Verdadero. $12 se setea en 1.
BEQ  $12, $0, 2     	# Si $12 == 0 saltaría. Como es 1, no salta (sigue a la sig. dirección).
BGT  $10, $11, 1    	# $10 (50) > $11 (0) -> Verdadero. Salta esquivando el ADDI (salta a 0x10).
ADDI $11, $11, 100  	# [Esta instrucción es salteada y no se ejecuta].
J	0x00000400     	# Salto incondicional a la dirección 0x0400 (0x100 en palabras).
# ... en la direccion 0x00000400 ...
JR   $20            	# Salta directamente a la dirección guardada en $20 (0x0400). Loop infinito.
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0xB2980064  # SLTI $12, $10, 100 (Opcode 22, rs=10, rt=12, imm=100)
0x00000004 : 0x83000002  # BEQ  $12, $0, 2	(Opcode 16, rs=12, rt=0, imm=2)
0x00000008 : 0x9A960001  # BGT  $10, $11, 1   (Opcode 19, rs=10, rt=11, imm=1)
0x0000000C : 0x0AD60064  # ADDI $11, $11, 100 (Opcode 1, rs=11, rt=11, imm=100)
0x00000010 : 0x10000100  # J	0x00000400	(Opcode 2, addr=0x100 en palabras)
0x00000400 : 0x0500000E  # JR   $20       	(Opcode 0, rs=20, rt=0, rd=0, func=14)
```

## Postcondiciones:
- Avanzar la CPU usando `step 5` para recorrer toda la secuencia lógica.
- Leer `$11`: debe mantenerse en `0` (lo cual comprueba que el branch `BGT` funcionó esquivando la suma).
- Leer el `$pc` (Program Counter) final: debe apuntar a `0x00000400` (comprobando que el salto incondicional y el salto por registro se completaron).

## Conclusiones:
Anduvo sin problemas. La resolución de los saltos relativos en instrucciones tipo I (`BGT`, `BEQ`) funcionan correctamente como offsets de palabras y no de bytes. El salto `J` dividió bien la dirección destino para ajustarse al formato de palabra, y el salto mediante registro `JR` permitió transferir el control a un espacio arbitrario validando los modos de direccionamiento.
