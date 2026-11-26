# PLANO — vla-v72 "VLA: robôs que veem, entendem e agem" (19/10/2026)

Formato INEMA Learning v7.2 · 8 aulas · PT (EN/ES depois, pela sessêo principal) · substitui https://inematds.github.io/vla/ (repo `~/projetos/vla`, intocado).
Fontes de fato: `~/projetos/wifi/longrun/2026-10-19-vla-v72/PESQUISA-MODELOS.md` e `PESQUISA-ECOSSISTEMA.md` (só [FATO]; [2ª]/[incerto] com ressalva) + docs oficiais abaixo.

## Aulas

| # | slug | objetivo (≤160) | diagrama | prática → pronto quando |
|---|---|---|---|---|
| 1 | cenario | distinguir LLM, VLM, VLA e World Action Model e dizer o que mudou de 2023 a out/2026 | escada LLM → VLM → VLA → WAM | classificar 6 sistemas reais → cada um com categoria + fonte |
| 2 | anatomia | explicar o caminho imagem + frase → ações e por que action chunking e flow matching existem | encoder → backbone → action head (bins / FAST / flow) + S2/S1/S0 | orçamento de tempo a 50 Hz com chunk de N → número bate com o gabarito |
| 3 | modelos | escolher um modelo para um caso real pela licença, tamanho e acesso (aberto, nêo comercial, API, early access) | mapa abertos × fechados | ler 2 model cards públicos e preencher a ficha → 2 campos de licença certos |
| 4 | dados | explicar de onde vêm os dados de robô e abrir um LeRobotDataset v3 com o LeRobot | funil teleoperaçêo → datasets abertos → vídeo humano → simulaçêo | carregar `lerobot/pusht` e imprimir episódios e fps → o comando imprime os dois |
| 5 | lerobot | instalar o LeRobot 0.6, treinar uma política pequena e entender gravar → treinar → rodar | ciclo record → train → eval/rollout | treino curto de ACT → existe checkpoint em `outputs/` |
| 6 | simulacao | avaliar uma política em simulaçêo e saber por que LIBERO sozinho nêo basta mais | sim → benchmarks → reality gap | `lerobot-eval` no PushT → sai a taxa de sucesso em N episódios |
| 7 | fronteira | explicar RL depois da imitaçêo, World Action Models e memória longa com exemplos de 2026 | imitaçêo → RL pós-treino → WAM → memória | simfila imitar → rodar → falhar → corrigir; aluno diz qual etapa reduziu a falha |
| 8 | mundo-real | ligar um VLA ao ROS 2, montar o checklist de segurança e traçar o seu próximo passo | ROS 2 Lyrical (nó da política → ros2_control) + camadas de segurança | mapa + gerador; checklist preenchido para um caso |

## Documentaçêo oficial lida (19/10/2026) — LeRobot v0.6.1 (Latest no GitHub, 03/08/2026)
- https://huggingface.co/docs/lerobot/installation (extras `dataset`, `training`, `hardware`, `viz`, `evaluation`, `pusht`/`aloha`; TorchCodec nêo existe em Linux aarch64 → cai para pyav)
- https://huggingface.co/docs/lerobot/il_robots (lerobot-record / lerobot-train / checkpoints em outputs/train/…/checkpoints)
- https://huggingface.co/docs/lerobot/lerobot-dataset-v3 (`from lerobot.datasets import LeRobotDataset`)
- https://github.com/huggingface/lerobot (README: `lerobot-train --policy.type=act`, `lerobot-eval --policy.path … --env.type … --eval.n_episodes`)
- https://huggingface.co/blog/lerobot-release-v060 (lerobot-rollout com estratégia dagger, LeLab, guia de hardware, HF Jobs)
- https://huggingface.co/docs/lerobot/evaluation → "nêo existe na v0.6.1" (usei o exemplo do README)
- https://github.com/NVIDIA/Isaac-GR00T (o backbone Cosmos-Reason2-2B é gated no Hub: precisa pedir acesso)

## Testes reais dos comandos (venv `.venv-teste`, fora do git)
(preenchido abaixo conforme rodar)
Máquina: NVIDIA GB10 (aarch64, 20 núcleos), Ubuntu, Python 3.12.3, `uv`. Sem token do HF: `HF_HOME` próprio (`teste/hf`) + `HF_HUB_DISABLE_IMPLICIT_TOKEN=1` (o Hub avisou "unauthenticated requests" — funcionou sem login).

| Comando | Resultado (19/10/2026) |
|---|---|
| `uv venv -p 3.12 .venv-teste` + `uv pip install "lerobot[training]==0.6.1"` | ok; torch 2.11.0+cu130, `torch.cuda.is_available()` = True |
| `LeRobotDataset("lerobot/pusht")` → `num_episodes`, `fps`, `num_frames` | 206 episódios, 10 fps, 25650 quadros (12,5 s com download) |
| `lerobot-train --policy.type=act --dataset.repo_id=lerobot/pusht --steps=200 --save_freq=200 --batch_size=8 --policy.device=cuda --policy.push_to_hub=false --wandb.enable=false --output_dir=… --job_name=…` | ok, 26 s no total; ACT com 52M parâmetros; checkpoint em `outputs/train/…/checkpoints/000200` e `last` |
| mesmo comando com `--policy.device=cpu` (CUDA_VISIBLE_DEVICES vazio) | ok, 321 s (~5 min, 1,2–1,6 s/passo) |
| idem 5000 passos, batch 32, GPU | 474 s; loss 0,329 |
| `uv pip install "lerobot[pusht,diffusion]==0.6.1"` | ok (gym-pusht 0.1.8, pymunk, pygame) |
| `lerobot-eval --policy.path=lerobot/diffusion_pusht --env.type=pusht …` | **FALHOU**: checkpoint antigo pede migraçêo ("requires migration to processor format") → nêo ensinado |
| `lerobot-eval` com `--eval.batch_size=10` sem `--eval.use_async_envs=false` | **FALHOU**: subprocessos nêo acham o namespace `gym_pusht` → ensinar com `--eval.use_async_envs=false` |
| `lerobot-train` sem `--policy.push_to_hub=false` (10 passos) | **FALHOU** como esperado: `ValueError: 'repo_id' argument missing` → a aula ensina o flag |
| `lerobot-eval` sem `lerobot[diffusion]` para a política diffusion | ImportError pedindo `lerobot[diffusion]` |
| `lerobot-eval --policy.path=outputs/train/act_pusht_teste/checkpoints/last/pretrained_model --env.type=pusht --eval.n_episodes=10 --eval.batch_size=10 --eval.use_async_envs=false --policy.device=cuda --output_dir=outputs/eval/act_200` | ok, 13 s: pc_success 0.0, avg_max_reward 0.082, 10 vídeos mp4 |
| idem com o checkpoint de 5000 passos, 20 episódios | ok, 18 s: pc_success 0.0, avg_max_reward 0.275 |

Decisões: prática 6 avalia a política que o aluno treinou (nêo há checkpoint público de PushT compatível testado); a aula ensina que sucesso 0% com recompensa subindo é resultado honesto de treino curto. SDL_VIDEODRIVER=dummy usado (máquina sem tela). LIBERO nêo testado (instalaçêo pesada), citado só como benchmark do `lerobot-eval`.

## Outras fontes oficiais lidas (19/10/2026)
- https://huggingface.co/lerobot/smolvla_base (raw README: `license: apache-2.0`, base `HuggingFaceTB/SmolVLM2-500M-Video-Instruct`)
- https://huggingface.co/nvidia/GR00T-N1.7-3B (NVIDIA Open Model License Agreement; 3.000.000.000 parâmetros; 13 datasets "Hybrid: Human, Robot, Simulated")
- https://huggingface.co/blog/nvidia/gr00t-n1-7 ("20,000+ hours of human egocentric video"; licença comercial)
- https://huggingface.co/datasets/lerobot/pusht (206 episódios, 10 fps, 1 câmera 96×96)
- Metadados testados: `lerobot/libero` 1693 episódios / 10 fps / 2 câmeras; `lerobot/aloha_mobile_cabinet` 85 / 50 fps / 3 câmeras.

## Desvios do CURRICULO
- Prática 6 avalia o checkpoint do próprio aluno no PushT (nêo LIBERO): `lerobot/diffusion_pusht` nêo carrega na 0.6.1 e LIBERO nêo foi instalado.
- Aula 7: além do simfila, um bloco `codigo` (ciclo.md + `lerobot-rollout --strategy.type=dagger` da doc, nêo rodado) porque o validador exige botêo copiar. simconversa opcional nêo feita.
- Comandos de robô (`lerobot-record`, `lerobot-rollout`) vêm da doc oficial e estêo marcados "nêo rodado aqui (precisa do braço)".
- Link do 8-K da Tesla (sec.gov) tirado: 403 no validador.
