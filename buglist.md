# BUGS ENCONTRADOS E MELHORIAS PROPOSTAS PÓS AVALIAÇÃO DOS BETA TESTERS
- Após criar novo usuário, não está indo para tela de personagens, fica travado na tela de login em loop; (Prioridade máxima)
- Equipar 2 armaduras é permitido - vamos criar slots (peito, luvas, botas, capacete, calças, mão esquerda, mão direita, capa, anéis(consultar max do dnd), e colar);
- Editar/excluir ações personalizadas - não tem nenhuma das duas operações implementadas;
- Pesquisa por texto quebrada (catálogo de itens e enciclopédia);
- Quando toma dano e tiver PV temporário, não sai dos PVs temporários, sai do hp verdadeiro;
- Armadura de placa parcial não atualiza CA - possivelmente coisa de itens legado não tendo mecanicas ativas;
- Descanso curto/longo não recuperando cargas;
- itens que dão set no valor ou adicionam bonus de atributo não funcionando (talvez seja coisa de item legado);
- Rolar inspiração bárdica não tá gastando a carga (quando usa no menu ações) -> investigar se é questão de feat legado;
- Canalizar divindade sem ação (não tem botão de rolar, usar, nadinha);
- Ao escolher antecedentes, as feats de origem não estão se limitando as feats de origem, da pra escolher qualquer uma;
- Quando personagem estiver vinculado a campanha, mestre decide se pode re-rolar atributos, na ficha de personagem básica permitir re-rolar;
- Pós compra de item, tiraria conversão automática; (pouco importante, pode postergar);

## PLANO DE AÇÃO
- [ ] Mapear os bugs que ocorreram em zonas próximas ou se tratam do mesmo assunto/pode ter ocorrido no mesmo setor do código/pode ser o mesmo bug;
- [ ] Agrupar esses bugs e criar uma spec de fix para cada um deles;
- [ ] Investigar a causa raiz desse bug particular;
- [ ] Investigar a repercussão desse bug no restante do código, onde pode ter bugs similares?
- [ ] Avaliar como código ou até mesmo dados legado podem ser afetados pelas mudanças feitas para correção dos bugs;
- [ ] Planejar adequação dos dos dados/códigos legado para não gerar discrepância do código em diferentes setores;

## TAREFA SECUNDÁRIA
- Crie uma skill no plugin sdd-scaffold-plugin para correção de bugs baseada nessa metodologia. Inclusive, crie a skill antes de executar o pipeline