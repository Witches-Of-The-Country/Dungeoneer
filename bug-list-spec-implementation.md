- Character wizard não permite escolha de feats/subclasses(quando começa acima do lv 1);
- Level up não permite mudar classe
- Método de ganho de pontos de vida é escolhido na criação de personagem do wizard e deve ser persistido no level up;
- Atributos/feats no modal de level up so me permite escolher +2 ASI ou +1 +1 ASI, não tem feats para escolher
- Quando começo um personagem acima do level 1, não tem o prompt de level up para que eu possa fazer minhas escolhas dos levels seguintes q estão pendentes; Quando um novo personagem tiver inicio acima do level 1, escolher somente classe principal e deixar a escolha de feats/multiclass/subclass para o wizard de level up
- Concluir level up falha por dados inválidos, segue response:
```json
{"timestamp":"2026-11-01T02:51:28.444Z","status":400,"error":"Bad Request","path":"/api/characters/36b37832-2d47-41bb-9147-40adbaa8fce1/level-up"}
```
- Papéis ativos no menu do perfil devem ser toggleables
- Menu de criação de feats/classes/itens homebrew continua com o modelo antigo, sem configuração de mecanicas reais;
- classes na enciclopédia continuam mostrando só aquelas infos básicas e não tá dando pra clicar para expandir mais infos