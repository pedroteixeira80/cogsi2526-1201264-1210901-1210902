# cogsi2526-1210901-1210902
## CA1
### Self-evaluation
Hélder Rocha (1210901) - 50%\
Pedro Teixeira (1210902) - 50%

## Technical Report

- ### Description of analysis
Analisamos os objetivos para a CA1 e percebemos que iria ser necessário utilizar vários comandos de git.

- ### Implementation
- 
#### Part 1
- 1 - Criamos o diretório CA1 (mkdir CA1) e copiamos o projeto spring-framework-petclinic para dentro do diretório CA1 (mv spring-framework-petclinic CA1/).
- 2 - Fizemos um git add . para adicionar todos os ficheiros ao stage.
- 3 - Fizemos um git commit -m "Setup CA1 project and initial Readme" para criar o primeiro commit.
- 4 - Fizemos push para o repositório remoto (git push origin).
- 5 - Criamos a primeira tag (git tag v1.1.0) e fizemos push da tag para o repositório remoto (git push origin v1.1.0).
- 6 - Criamos um novo atributo na classe Vet (private int professionalLicenseNumber) e o seu respetivo suporte.
- 7 - Foi necessário alterar alguns testes para garantir compatibilidade com este novo atributo.
- 8 - Utilizamos o comando git commit -am "Added and tested professional license number to vet" para criar um novo commit (este comando engloba o "git add ." e o "git commit -m ".."").
- 9 - Fizemos push do commit para o repositório remoto (git push origin).
- 10 - Criamos uma tag para este commit (git tag v1.2.0) e fizemos push da tag para o repositório remoto (git push origin v1.2.0).
- 11 - Utilizamos os comando git log com as opções --oneline e --graphgit.
- 12 - Testamos fazer revert para um commit anterior (git revert 6f293bc).

#### Part 2
- 13 - Criamos uma nova branch para desenvolvermos a feature pedida (git switch -C feature/email-field)
- 14 - Criamos um novo atributo na classe Vet (private int email) e o seu respetivo suporte.
- 15 - Foi necessário alterar alguns testes para garantir compatibilidade com este novo atributo.
- 16 - Utilizamos o comando git commit -am "Added email field to the Vet" para criar um novo commit (este comando engloba o "git add ." e o "git commit -m ".."").
- 17 - Demos push do commit para a nossa branch remota (git push origin feature/email-field).
- 18 - Fizemos checkout para a nossa branch main para podermos dar merge da nossa branch de desenvolvimento com main (git checkout main).
- 19 - Depois de estarmos na branch main fizemos o merge (git merge feature/email-field).
- 20 - O merge foi bem sucedido pois não haviam quaisquer conflitos.
- 21 - Depois do merge podemos verificar que o commit que estava na nossa branch de desenvolvimento agora se encontra em main.
- 22 - Criamos uma tag v1.3.0 em main com o nosso commit que continha as alterações. 
- 23 - De maneira a criar conflitos para se realizar um rebase, criei duas nova branch (git switch -C test_conflicts_1 e git switch -C test_conflicts_2) e alterei a linha 11 do readme em cada uma das branches.
- 24 - Repeti o que foi feito nos steps 16,17,18,19.
- 25 - Numa das branchs o merge correu bem como o anterior, mas na outra deparamo-nos com conflitos.
- 26 - Para resolver estes conflitos, fomos à branch main (git checkout main) e fizemos git pull de maneira a ficarmos com esta branch atualizada.
- 27 - Depois fomos para a nossa branch de desenvolvimento (git checkout test_conflicts_2) e fizemos um rebase com a branch main (git rebase main).
- 28 - Surgiram conflitos que foram tratados no código.
- 29 - Demos git add ao readme para atualizá-lo, de seguida fizemos git rebase --continue e verificamos que não haviam mais conflitos, tal como esperado.
- 30 - Depois repetimos os passos 18,19 e 20 que já correram como esperado pois o rebase já tinha sido realizado.