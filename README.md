# opcodesDB:
opcodesDB is a CPU low level environment representation (registers, flags, instructions, ...).  Data are listed in a packed dynamic structure which can be unpacked by a parser at any time.

This project is a fruit of many years of development and a lot of attempts to standardize CPU environment.

Currently, only two architecture are supported (x86 and x64).

# Using opcodesDB:
If you’re a Perl developer, you can directly start exploring x86 environment by calling ```getEnvironment``` function:
```
require 'x86.pl';
my $environment = getEnvironment();
# explore ...
```
If not, that’s not a problem, all what you need to do is to export the environment to a JSON file and use your favorite programming language to parse the environment as a pure JSON:
```
# open a cmd in the opcodesDB folder and run this command:
export.pl x86.json
```
## X86 environment:
-	**name** = environment name.
-	**version** = environment version.
-	**architectures** = x86 architectures (x86,x64 or both ‘x86-64’).
-	**registers** = contains a hash of all registers used by x86.
-	**flags** = contains all registers flags (mxcsr, x87cw, x87sw,(e|r)flags).
-	**instructions** = contains a list of all instructions used by x86 environment (Intel and AMD).

### Instructions:
Each instruction is represented as a hash and contains the following info:
-	**mnemonic**: instruction mnemonic (name).
-	**architecture**: required architecture for instruction.
-	**deprecated**: if true, it means that manufacturing (Intel or AMD) stopped supporting such instruction.
-	**bnd|rep|repe|repne**: if true, instruction supports that prefix.
-	**level**: privilege level required to execute the instruction (3 or 0).
-	**cupid**: a list of required flags of cupid to run this instruction.
-	**suppressAllExceptions**: AVX512 instruction supports suppressing all exceptions {sae}.
-	**embeddedRounding**: AVX512 instruction supports embedded rounding {er}.
-	**fpuPush|fpuPop** = N: FPU instruction pushes|popes stack-register N time.
- **fpuTop** = N: FPU instruction increment top register by N time (N could be negative = decrement).
-	**lock**: provides lock info for the instruction. Instruction is lockable if **hardware|legacy** is set.
	  - **hardware** : hardware lock is supported.
	  - **legacy** : locking using legacy lock prefix is supported. 
	  - **implied** : instruction is lockable either with or without the presence of the lock.
	  - **explicit** : lock prefix is required for hardware lock.
	  - **ignore** : accept lock but ignore it because memory is not the destination operand.
-	**opcode**: is a list of hash representing the instruction opcode. Each hash has a name (evex,vex,xop,3dnow,opcode,modrm,imm…) and a size. Depending on hash name, it contains field specified to that name.
    - **vex|evex|xop**:
    	- size: vector prefix size (2|3|4).
      - length: vector length (128|256|512).
      - mmmmm: vector embedded escaping (0f|0f38|0f3a|m8|m9).
      - pp: vector.pp field (mandatory prefixes).
      - vvvv: vector.vvvv field (nds|ndd|dds).
      - w: like rex.w (1|0|ignore).
    - **prefix**
      - value = decimal code for prefix (0x66, 0xf2, 0xf3).
    - **escape**:
      - value: table escape opcode.
    - **opcode**:
      - value: opcode value in decimal.
      - reg: if argument is encoded inside the opcode, this field describes the register (i => fpu, r[bwdq] => general reg).
    - **modrm**:
      - reg: if defined, it indicates that ModRm.Reg must be equal to reg.
    - **imm|offset|moffs|3dnow**: standar fields. they have size and value fields.
    
-	**operands**: a list of hash representing arguments used by the instruction.
  -	**optional**: if true, it means that this argument is optional and assembler/dissembler may omit it.
  -	**embedded**: argument is built-in instruction and it does not require encoding.
  -	**encoding**: info about how to encode/decode this argument.
  -	**value**:  if(Embedded) it contains register name or immediate value.
  -	**read**: if true, the operand is read.
  -	**write**: if true, the operand is written.
  -	**size**: argument size in bits.
  -	**type**: argument type (reg, imm,…).
  -	**mem**: if defined, it means that argument is a memory or supports memory addressing. It contains the following info:
    - size : size of memory in bits.
    - seg: memory segment.
    -	index: memory index register.
    -	base: memory base register.
    - tuple: AVX512 tuple used to encode/decode compressed displacement (DISP8*N).
    - brdcst : AVX512 memory broadcast size.
    - vsibSize: AVX vsib size (128|256|512).
    - type: memory type (m8,m128, ptr16,m16-m32,...).

-	**Flags**: modified flags by instruction. Each flag is represented as follow :
  - **T** = instruction Tests flag.
  - **M** = instruction Modifies flag.
  - **C** = instruction sets flag to zero (Clear).
  - **S** = instruction Sets flag to 1 (Set).
  - **U** = Undefined.
  - **N** = Not affected.
  - **X** = TM.

## Example:
Consider this instruction: ```'evex.nds.512.0f.w0 58 /r' 'vaddps zmm {k} {z}, zmm, zmm/m512/b32 {er}'``` . After parsing it, the parser reports the following result:
```
{
	'architecture'          => 'x86-64',
	'mnemonic'              => 'vaddps',
	'level'                 => 3,
	'deprecated'            => 0,
	'fpuPush'               => 0,
	'fpuPop'                => 0,
	'fpuTop'                => 0,
	'rep'                   => 0,
	'repe'                  => 0,
	'repne'                 => 0,
	'suppressAllExceptions' => 0,
	'embeddedRounding'      => 1,
	'bnd'                   => 0,
	'cpuid'                 => ['AVX512F'],
	'lock'                  => {
		'hardware' => 0,
		'legacy'   => 0,
		'implied'  => 0,
		'explicit' => 0,
		'ignore'   => 0
	},

	'operands' => [
		{
			'value'    => undef,
			'encoding' => 'modrm.reg',
			'read'     => 0,
			'embedded' => 0,
			'mem'      => undef,
			'type'     => 'zmmreg',
			'zeroing'  => 1,
			'mask'     => 1,
			'size'     => 512,
			'write'    => 1,
			'hint'     => 0,
			'optional' => 0
		},
		{
			'mem'      => undef,
			'type'     => 'zmmreg',
			'embedded' => 0,
			'value'    => undef,
			'encoding' => 'evex.vvvv',
			'read'     => 1,
			'optional' => 0,
			'write'    => 0,
			'size'     => 512
		},
		{
			'embedded' => 0,
			'type'     => 'zmmreg',
			'mem'      => {
				'tuple'    => 'fv',
				'index'    => undef,
				'size'     => '512',
				'base'     => undef,
				'seg'      => undef,
				'brdcst'   => '32',
				'type'     => 'm512',
				'vsibSize' => undef
			},
			'value'    => undef,
			'encoding' => 'modrm.rm',
			'read'     => 1,
			'optional' => 0,
			'size'     => 512,
			'hint'     => 1,
			'write'    => 0
		}
	],
	'opcode' => [
		{
			'length' => '512',
			'mmmmm'  => '0f',
			'name'   => 'evex',
			'size'   => 4,
			'vvvv'   => 'nds',
			'w'      => 0
		},
		{
			'reg'   => undef,
			'name'  => 'opcode',
			'size'  => 1,
			'value' => 88
		},
		{
			'reg'  => undef,
			'name' => 'modrm',
			'size' => 1
		}
	],
	'flags'    => {},
	'mxcsr'    => {},
	'x87flags' => {
		'sw' => {},
		'cw' => {}
	  }

};
```

