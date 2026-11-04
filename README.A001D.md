# Kernel 6.19.5 para TWRP no ASUS ZB634KL (ASUS_A001D)

Port do [msm8953-mainline](https://github.com/msm8953-mainline/linux), branch `6.19.5/main`, para
uso no TWRP do Zenfone Max Shot / Max Plus M2. As alterações estão no branch `a001d-twrp`.

Este kernel é feito para uma recovery. **Não serve para rodar o Android do aparelho.**

Use o `6.19.5/main`, não o `master`, que está parado no 6.19.0 e não tem os drivers de carregador e
bateria do PMI8950.

## O que foi alterado

| Área | Alteração |
|---|---|
| DTS | `sdm450-asus-a001d.dts`, com tensões, memória reservada e ligações da stock |
| Painel | `panel-tcl-ft8719.c`, gerado da DTB stock |
| Bateria | `pmi8994_fg`: lê o status do carregador ao iniciar |
| Config | `a001d_twrp_defconfig`, com tudo que a recovery usa embutido |
| Imagem | Alvo `Image.gz-dtb` em `arch/arm64/Makefile`, porque o LK stock só inicia com a DTB anexada |

## Requisitos deste aparelho

Cada item abaixo já impediu o boot ou deixou o aparelho inutilizável:

| Requisito | O que acontece sem ele |
|---|---|
| `Image` bem abaixo de 50 MiB (hoje ~27 MiB) | Trava na logo. O LK põe o kernel em `0x40080000` e o ramdisk em `0x43600000` |
| Reserva de `0x84a00000` a `0x86800000` | Área protegida pelo TrustZone |
| `s3` em 1,224 V, não nos 1,225 V da stock | O valor não existe no driver, e **todos** os reguladores falham: sem eMMC, tela e USB |
| `MAILBOX`, `QCOM_APCS_IPC`, `RPMSG_QCOM_SMD` embutidos | Sem comunicação com o RPM, sem reguladores |
| `QCOM_CLK_APCS_MSM8953`, `QCOM_CPR4PD` e cpufreq | CPU presa em 748,8 MHz e interface lenta |
| `IPV6=y` | O `adbd` aborta |
| `sys.use_memfd=true` (device tree) | Este kernel não tem `ashmem`, e o `recovery` aborta |
| `FRAMEBUFFER_CONSOLE` desligado | O console do kernel desenha por cima do TWRP |

## Compilar

Requer `gcc-aarch64-linux-gnu`. Os compiladores das árvores AOSP são antigos demais para este kernel.

O kernel é compilado pelo `make recoveryimage` do TWRP, com `A001D_KERNEL=6.19`. Veja o README do device tree em
`device/asus/A001D`.

O `check_kernel.sh` falha se alguma opção pedida como `=y` não for aplicada.

## Testado

Com TWRP via `fastboot boot`, em um ASUS_A001D_1 com painel TCL: boot, eMMC em HS400, tela, touch,
adb, MTP no Windows, porcentagem e status da bateria, limites de carga e CPU a 1,8 GHz.

## Limitações

- **`reboot bootloader` vai para o Android.** Use Energia + Volume para cima. Gravar o motivo no
  `imem`, reset morno, `PS_HOLD` direto e chamada SCM não resolveram, e esse código foi revertido.
  `Reboot` → `Recovery` provavelmente tem o mesmo problema, mas não foi testado.
- **Sem descriptografia da `/data`.** Este kernel não tem o caminho de FDE da stock
  (`qseecom`, `dm-req-crypt`). Não é um ajuste simples de config.
- **Só o painel TCL.** Em aparelhos com painel Boyi ou Holitech, **provavelmente não há imagem**,
  porque os dois não têm driver aqui.
- **Carga:**
  - Limitada a 1 A, contra 1,85 A da stock, porque não há redução por temperatura.
  - Os limites de temperatura da stock não são aplicados.
  - Com porta USB comum, a entrada fica em 500 mA.
  - Não deixe carregando sem supervisão.
- **Porcentagem da bateria:** pode ser menos precisa, porque o driver não usa o perfil da ASUS.
- **GPIOs reservados:** a lista é uma estimativa baseada em outros telefones msm8953, porque a do
  A001D não é conhecida. Ler um GPIO protegido pode reiniciar o aparelho.
- **Boot mais lento:** o adb sobe em ~45 s, contra ~17 s no 4.9.
- **Cópia em memória:** uma única medição mostrou piora com o ajuste de frequência ligado, sem causa
  conhecida. Na prática, o desempenho da interface ficou comparável ao do 4.9.
- **Avisos no log, sem efeito observado:**
  - `clk-rcg2: update_config` ao iniciar o painel.
  - `cpu0: Failed to set OPP config` no início do boot.
  - `grep: Invalid argument`, do `toybox` do TWRP, ao ler pipes e `/proc`; o resultado continua certo.
- **Sem suporte:** câmera, áudio, modem e Wi-Fi. A config desliga esses blocos ou não os testa.
- **Não testado:** microSD, USB OTG, suspensão prolongada e instalação na partição `recovery`.
- **Histórico:** o clone é raso (`--depth=1`). Rode `git fetch --unshallow` antes de publicar.

## Sem USB? Como ler o log

`ramoops` em `0xef000000`, com as mesmas posições no 4.9 e no 6.19:

1. Com o aparelho travado no 6.19, segure Energia + Volume para cima até o fastboot.
2. Rode `fastboot boot` no TWRP 4.9.
3. Leia `/sys/fs/pstore/console-ramoops-0`.

Não passe pelo Android antes de ler, porque ele sobrescreve essa área da RAM.
