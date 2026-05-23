<!-- antes  de enviar a versão final, solicitamos que todos os comentários, colocados para orientação ao aluno, sejam removidos do arquivo -->
# Análise comparativa de modelos OCR e técnicas de pré-processamento na extração de dados de documentos de identidade brasileiros

#### Aluno: [Lucas Emanuel Ferreira Ramos](https://github.com/luckermos)
#### Orientadora: [Manoela Rabello Kohler](https://github.com/manoelakohler)

---

Trabalho apresentado ao curso [VC MASTER](https://ica.puc-rio.ai/vc-master) como pré-requisito para conclusão de curso e obtenção de crédito na disciplina "Projetos de Sistemas Inteligentes de Apoio à Decisão".

- [Link para o código](ProjetoFinal.ipynb).

---

### Resumo

O trabalho avalia o impacto de técnicas de pré-processamento de imagem na extração de textos de RGs brasileiros usando três motores de OCR: Tesseract, Easy OCR e DocTR. O foco principal é medir a eficácia da extração textual global por meio de busca difusa (Distância de Levenshtein), sem a necessidade de classificar ou separar os campos do documento.

O aumento de resolução (Upscaling) foi a única técnica que trouxe melhorias consistentes e universais para todos os três modelos quando aplicada isoladamente.

Motores modernos baseados em Deep Learning e Transformers (EasyOCR e DocTR) dispensam pipelines complexos de tratamento de imagem. O DocTR se consolidou como o melhor modelo do ecossistema avaliado.

Embora ultrapassado para imagens puras, o Tesseract ainda se mostra competitivo desde que amparado por um pipeline cirúrgico de tratamento (Upscaling + Non-Local Means).

### 1. Introdução

Neste projeto é explorada a aplicação de modelos de OCR (*Optical Character Recognition*) em documentos reais de identificação brasileiros (RG) com a utilização de diferentes abordagens de pré-processamento das imagens, isolados e combinados: Upscaling, Remoção de Ruído, Aumento de Nitidez, CLAHE e Binarização Otsu.

O principal objetivo é identificar as abordagens de pré-processamento que têm melhor desempenho em cada modelo, com foco na extração das informações, sem se preocupar com a distinção e identificação dos campos.

Modelos pré-treinados (Tesseract OCR, Easy OCR e DocTR) serão utilizados juntamente com diferentes pipelines de pré-processamento para identificar as melhores combinações baseado em uma métrica de avaliação proposta que utiliza busca difusa, isto é, uma avaliação baseada em busca de conteúdo (usando distância de levenshtein), priorizando avaliar se o modelo conseguiu extrair a informação, mesmo sem distinguí-la.

### 2. Modelagem

Os modelos foram avaliados aplicando-os nas imagens originais e comparando o resultado com as diferentes técnicas de pré-processamento que atacam problemas clássicos na leitura de documentos. Cada técnica será aplicada de forma isolada e, também, de forma composta, incrementando cada camada de tratamento para tentar absorver o melhor de cada uma delas sem atrapalhar as técnicas seguintes.

**Técnicas Isoladas**

1. *Upscaling* com Lanczos

  - O redimensionamento (*upscaling*) é vital quando o texto do documento é muito pequeno para o modelo. O algoritmo escolhido, Lanczos, utiliza uma função para calcular os novos pixels. Isso resulta em uma reconstrução muito mais nítida das bordas, com menos serrilhado (*aliasing*) e sem o efeito "embaçado" de outros métodos. A resolução será ampliada em 2x.

2. Redução de Ruído com *Non-Local Means*

  - Documentos costumam ter fundos complexos (marcas d'água e microimpressões). Filtros comuns podem borrar o texto junto com o ruído. O *Non-Local Means* (NL-Means) identifica que o "ruído" é aleatório, enquanto os traços das letras são padrões estruturados, removendo a granulação sem prejudicar os caracteres.

3. Aumento de Nitidez com *Unsharp Masking*

  - A nitidez é aumentada removendo as frequências baixas e mantendo apenas as bordas e os contornos das letras (*Unsharp Masking*). Em fotos de celular, as bordas das letras pequenas (como a data de nascimento ou órgão emissor) costumam ser suaves demais. O *Unsharp Masking* aumenta o contraste local nessas bordas, tornando a transição entre a letra e o fundo muito mais visível.

4. Ajuste de Iluminação com CLAHE

  - É muito comum fotos de RG terem sombras de um lado ou reflexos de flash no centro. O CLAHE consegue clarear as áreas escuras e reduzir o brilho das áreas claras localmente, uniformizando o fundo para que o texto se destaque por igual em todo o documento.

5. Binarização com Otsu
  
  - A binarização Otsu calcula automaticamente o melhor limiar para separar o que é fundo (branco) do que é texto (preto). O algoritmo Otsu consegue definir com precisão o que é traço de letra e o que é papel, gerando uma imagem puramente em preto e branco, o que pode facilitar a extração por parte do modelo.

**Técnicas Compostas**

A primeira camada aplicada será o ***Upscaling* com Lanczos**. Aumentar primeiro a resolução fará com que todas as técnicas subsequentes trabalhem com mais informação.

6. \+ Redução de Ruído com *Non-Local Means*

  - Remove o ruído e granulações que imagens complexas como documentos de identidade costumam ter.

7. \+ Aumento de Nitidez com *Unsharp Masking*

  - Recupera a nitidez das bordas das letras que a remoção de ruído possa ter suavizado.

8. \+ Ajuste de Iluminação com CLAHE

  - Já com uma imagem mais limpa, o CLAHE prepara o histograma da imagem aumentando o contraste das letras e das bordas reais.

9. \+ Binarização com Otsu

  - Transformação da imagem em Preto e Branco puro como etapa final antes de enviar ao motor de OCR.
  - A binarização deve ser a última camada, pois descarta informações de cores/cinza que as técnicas anteriores precisam.

**Observação**:

Na composição das camadas de pré-processamento acima, existem divergências na literatura acerca da posição ideal do CLAHE.

Na ordem considerada clássica, o CLAHE viria antes da redução de ruído e aumento de nitidez, com o objetivo de evidenciar os ruídos para que o NL-Means e *Unsharp Masking* sejam mais eficientes na limpeza da imagem.

Entretanto, o CLAHE aumenta o contraste local, e se a imagem original tem muito ruído - que é o caso de imagens com fundo complexo como o RG -, o CLAHE acaba por aumentar o contraste deles, tornando o ruído muito mais agressivo e difícil de remover depois.

Ao aplicar a Redução de Ruído (NLM) antes, o fundo é suavizado, removendo os "granulados" enquanto a imagem ainda tem baixo contraste. Após isso, o Unsharp Masking ajuda a definir oas bordas das letras, garantindo que o CLAHE tenha estruturas sólidas para realçar.

Portanto, neste caso de uso, o CLAHE foi aplicado somente após as camadas de Remoção de Ruído e Aumento de Nitidez.

### 3. Resultados

1. Tesseract OCR

Mantido pelo Google, é um dos modelos de OCR de código aberto mais famosos do mundo. Ele é baseado em uma arquitetura híbrida que combina processamento de imagem tradicional com Redes Neurais Recorrentes (LSTM).

O Tesseract é extremamente sensível ao fundo. Ele foi projetado para ler documentos escaneados (preto no branco) e não fotos de celular. Portanto, é esperado que os melhores resultados sejam obtidos após várias camadas de pré-processamento, transformando a imagem em preto e branco e sem ruídos.

O resultado do Tesseract com a imagem original foi bem ruim (64,3%, em média), conforme esperado. Aplicando as técnicas isoladamente, a única que melhorou o resultado foi o Upscaling (89,0%). Um simples aumento de resolução melhorou consideravelmente a assertividade.

Compondo as técnicas em camadas, o melhor resultado médio foi da combinação de Upscaling + Redução de Ruído (92,6%). Surpreendentemente as primeiras camadas já atingiram os melhores resultados, as camadas CLAHE e Otsu aplicadas no final acabaram por piorar os índices obtidos com as primeiras camadas.

2. Easy OCR

O EasyOCR é uma biblioteca moderna baseada em Deep Learning (PyTorch). Ele lida melhor com fotos do que o Tesseract.

O modelo divide a tarefa em duas partes:

Detecção: Usa o algoritmo CRAFT para localizar onde há texto na imagem.

Reconhecimento: Usa uma rede CRNN (Convolutional Recurrent Neural Network) para ler o conteúdo.

É um modelo mais robusto contra rotações, ruídos e variações de iluminação. Teoricamente não seria necessário muito processamento para obter bons resultados.

O Easy OCR obteve bons resultados em todas as abordagens (geralmente acima de 83%) inclusive com as imagens originais. Mostra que é um modelo mais robusto e preparado para lidar com fotos, comparado ao Tesseract.

Mais uma vez a aplicação do Upscaling, de forma isolada, aumentou o índice médio mais do que qualquer outra técnica por si (88,7%).

Ao compor as técnicas, houve ganho sucessivo até a camada Upscaling + Ruído + Nitidez, obtendo o melhor resultado do modelo (88,9%). A aplicação do CLAHE e Otsu fez com que o índice caísse levemente (abaixo de 88,6%).

Dessa forma, o modelo performou melhor com ajustes mais finos, sem a necessidade de binarização ou CLAHE.

3. DocTR

Desenvolvido pela Mindee, o DocTR é focado especificamente em documentos, sendo uma das ferramentas mais poderosas da atualidade para visão computacional aplicada.

Ele utiliza modelos de ponta baseados em Transformers (como o ViT - Vision Transformer) ou arquiteturas avançadas como ResNet/DBNet para detecção.

Custuma apresentar altíssima acurácia em documentos complexos (como RGs e CNHs).

Pode se beneficiar bastante do Upscaling para identificar os caracteres pequenos em RGs.

Já o CLAHE e o Otsu podem acabar descartando informações que o modelo considera importante para identificar os caracteres.

Este modelo, teoricamente, é o que menos precisaria de processamento para ter bons resultados.

A arquitetura robusta e moderna do DocTR fez com que o resultado fosse excelente, independente da abordagem utilizada (geralmente acima de 94%). O pior resultado, mas ainda bom, foi obtido aplicando somente a binarização Otsu às imagens (88,8%), possivelmente por descartar informações que o modelo considera importante.

Nota-se que o modelo se saiu muito bem com as imagens originais (95,4%). Esse desempenho é esperado, uma vez que o modelo foi projetado para ser end-to-end.

Ao aplicar as técnicas de pré-processamento, seja de forma isolada ou combinadas, o resultado pouco se altera. O modelo já sabe lidar com uma imagem imperfeita. Ao tentar limpá-la, pode-se acabar gerando artefatos que o modelo não reconhece como naturais, causando confusão.

O melhor resultado geral obtido foi com a aplicação isolada do Upscaling (95,8%). Mas que não foi muito diferente do obtido com as imagens originais.

O segundo pior resultado foi obtido ao combinar todas as camadas de processamento (92,5%).

### 4. Conclusões

Neste trabalho foi possível explorar a extração de informações de documentos de identidade brasileiros (RG) através da aplicação de 3 modelos de OCR (Tesseract OCR, Easy OCR e DocTR) em diferentes abordagens de pré-processamento isoladas e compostas: Upscaling, Remoção de Ruído, Aumento de Nitidez, CLAHE e Binarização Otsu.

O Tesseract OCR foi o modelo que obteve os piores índices de assertividade (vários deles abaixo de 65% na métrica de avaliação proposta), entretanto, com a aplicação das técnicas de Upscaling + Redução de Ruído, o resultado melhorou consideravelmente, atingido um índice de 92,6%.

Já o Easy OCR teve bons resultados em todas as abordagens (geralmente acima de 83%), sofrendo pouca alteração entre elas seja de forma isolada ou composta. Contudo, seu melhor resultado (Upscaling + Ruído + Nitidez, com índice 88,9%) não conseguiu bater o melhor índice do Tesseract.

O DocTR obteve os melhores resultados. Mesmo sem pré-processamento, o índice já ficou acima de 95%. Assim como no Easy OCR, as abordagens de pré-processamento pouco alteraram o índice. O melhor resultado obtido foi com a aplicação isolada do Upscaling, chegando a 95,8%.

Importante destacar o impacto do Upscaling, que trouxe melhoria nos índices para todos os modelos.

Portanto, dependendo do modelo de OCR escolhido devem ser consideradas diferentes técnicas de pré-processamento ou, até mesmo, nenhuma. Modelos mais modernos como EasyOCR e DocTR é possível obter bons resultados mesmo sem pré-processamento, com o DocTR - baseado em Trasformers - sendo consideravelmente superior. Já para o Tesseract, somente com a aplicação do Upscaling + Redução de Ruído foi possível obter índice melhor do que o EasyOCR e próximo, mais inferior, ao DocTR.

---

Matrícula: 232.100.383

Pontifícia Universidade Católica do Rio de Janeiro

Curso de Pós Graduação *Visão Computacional Master*
