# Guia do Sistema de Diálogo em GameMaker Studio

Este documento descreve um sistema de diálogo flexível e modular para GameMaker Studio 2 (GMS2), projetado para ser facilmente adaptado por qualquer desenvolvedor. O sistema suporta diálogos paginados, opções de escolha com ramificação, efeitos visuais no texto (como tremor, cores e animações), suporte a múltiplos idiomas, retratos de personagens e integração com o jogador. Ele é neutro em relação a assets, permitindo que você use seus próprios sprites, sons e fontes.

O guia é estruturado para ser claro e prático, com instruções para implementação e exemplos de código em GML (GameMaker Language). Ele assume que você configurará os assets (sprites, sons, fontes) no editor do GMS2 conforme necessário.

## Visão Geral

- **Funcionamento Básico**: O jogador interage com NPCs próximos pressionando uma tecla ou botão de gamepad, criando uma instância de `obj_dialogo` que carrega textos de um script centralizado (`scr_dialogues` ou equivalente para outros idiomas).
- **Recursos Principais**:
  - Diálogos paginados com revelação gradual de texto (caractere por caractere).
  - Efeitos visuais no texto via tags (ex.: `[tremor]`, `[cor=red]`).
  - Opções de escolha com diálogos ramificados.
  - Suporte a retratos de personagens à esquerda ou direita.
  - Pausa o jogo durante diálogos.
  - Fácil adição de novos diálogos via switch-case.
- **Variáveis Globais**:
  - `global.language`: Idioma atual (ex.: "en" para inglês, "pt-br" para português).
  - `global.is_game_paused`: Pausa o jogo.
  - `global.dialogue_active`: Indica se um diálogo está ativo.
  - `global.controller`: Índice do gamepad (se aplicável).
- **Requisitos**:
  - GameMaker Studio 2+.
  - Bibliotecas padrão (sem dependências externas).
  - Configure sprites, sons e fontes no editor (ex.: `spr_portrait_default`, `snd_dialogue_default`, `fnt_dialogue`).

## Passo 1: Configuração de Objetos Base

### obj_par_npc (Objeto Pai para NPCs)

- **Propósito**: Objeto pai para todos os NPCs, usado para herança e detecção de proximidade.
- **Criação**:
  - Crie um objeto vazio chamado `obj_par_npc` no GMS2.
  - Marque como "Persistent" se desejar que persista entre rooms (opcional).
  - Não adicione código diretamente; ele serve como referência.

### NPCs Individuais

- **Propósito**: Cada NPC é um filho de `obj_par_npc` e define um identificador de diálogo via variável `dialogue_id`.
- **Exemplo**:
  - Crie um objeto (ex.: `obj_npc_example`).
  - Defina `obj_par_npc` como pai no editor.
  - No evento **Create**:

    ```gml
    dialogue_id = "example_dialogue";  // Identificador único para o diálogo
    ```
- **Dica**: Coloque instâncias de NPCs no room editor. O jogador detectará proximidade via `distance_to_object`.

## Passo 2: Objeto de Diálogo (obj_dialogo)

Este objeto gerencia a exibição, progressão e efeitos do diálogo. Ele usa uma `ds_grid` para armazenar informações e suporta parsing de tags para efeitos visuais.

### Evento Create

Define estruturas de dados e inicializa variáveis.

```gml
enum DialogueInfo {
    Text,
    Portrait,
    Side,
    Name
}

npc_dialogue_id = "";
dialogue_grid = ds_grid_create(4, 0);
current_page = 0;
option[0] = "";
option_response[0] = "";
option_count = 0;
selected_option = 0;
draw_options = false;
is_initialized = false;
current_char = 0;
alarm[0] = 1;
words = [];
effects = [];
```

### Evento Cleanup

Libera recursos para evitar memory leaks.

```gml
ds_grid_destroy(dialogue_grid);
```

### Evento Step

Inicializa o diálogo, parseia texto com tags de efeitos, avança páginas e gerencia opções.

```gml
if (!is_initialized) {
    // Carrega o script de diálogos baseado no idioma
    if (global.language == "pt-br") script_execute(scr_dialogues);
    else script_execute(scr_dialogues_en);
    is_initialized = true;
    alarm[0] = 1;
    global.is_game_paused = true;
    
    // Parsear texto para palavras e efeitos
    var _text = dialogue_grid[# DialogueInfo.Text, current_page];
    words = [];
    effects = [];
    var _current = "";
    var _effect = { type: "none", color: c_white };
    var _in_tag = false;
    var _tag = "";
    
    for (var i = 1; i <= string_length(_text); i++) {
        var _char = string_char_at(_text, i);
        if (_char == "[") {
            _in_tag = true;
            _tag = "";
            continue;
        }
        if (_char == "]" && _in_tag) {
            _in_tag = false;
            if (_tag == "noeffect") _effect = { type: "none", color: c_white };
            else if (string_pos("shake", _tag) == 1) _effect.type = string_pos("/shake", _tag) == 0 ? "shake" : "none";
            else if (string_pos("bounce", _tag) == 1) _effect.type = string_pos("/bounce", _tag) == 0 ? "bounce" : "none";
            else if (string_pos("pulse", _tag) == 1) _effect.type = string_pos("/pulse", _tag) == 0 ? "pulse" : "none";
            else if (string_pos("wave", _tag) == 1) _effect.type = string_pos("/wave", _tag) == 0 ? "wave" : "none";
            else if (string_pos("tremor", _tag) == 1) _effect.type = string_pos("/tremor", _tag) == 0 ? "tremor" : "none";
            else if (string_pos("float", _tag) == 1) _effect.type = string_pos("/float", _tag) == 0 ? "float" : "none";
            else if (string_pos("zoom", _tag) == 1) _effect.type = string_pos("/zoom", _tag) == 0 ? "zoom" : "none";
            else if (string_pos("color=", _tag) == 1) {
                var _color = string_replace(_tag, "color=", "");
                if (_color == "red") _effect.color = c_red;
                else if (_color == "blue") _effect.color = c_blue;
                else if (_color == "yellow") _effect.color = c_yellow;
                else if (_color == "purple") _effect.color = c_purple;
                else if (_color == "orange") _effect.color = c_orange;
                else if (_color == "green") _effect.color = c_green;
                else if (_color == "/color") _effect.color = c_white;
            }
            continue;
        }
        if (_in_tag) {
            _tag += _char;
            continue;
        }
        if (_char == " " || i == string_length(_text)) {
            if (i == string_length(_text)) _current += _char;
            if (_current != "") {
                array_push(words, _current);
                array_push(effects, { type: _effect.type, color: _effect.color });
                _current = "";
            }
            if (_char == " ") {
                array_push(words, " ");
                array_push(effects, { type: "none", color: c_white });
            }
        } else _current += _char;
    }
}

// Avançar texto ou página
if (current_char < string_length(dialogue_grid[# DialogueInfo.Text, current_page])) {
    if (keyboard_check_pressed(vk_space) || gamepad_button_check_pressed(global.controller, gp_face1)) {
        current_char = string_length(dialogue_grid[# DialogueInfo.Text, current_page]);
    }
} else {
    if (current_page < ds_grid_height(dialogue_grid) - 1) {
        if (keyboard_check_pressed(vk_space) || gamepad_button_check_pressed(global.controller, gp_face1)) {
            alarm[0] = 1;
            current_char = 0;
            current_page++;
            var _text = dialogue_grid[# DialogueInfo.Text, current_page];
            words = [];
            effects = [];
            var _current = "";
            var _effect = { type: "none", color: c_white };
            var _in_tag = false;
            var _tag = "";
            for (var i = 1; i <= string_length(_text); i++) {
                var _char = string_char_at(_text, i);
                if (_char == "[") {
                    _in_tag = true;
                    _tag = "";
                    continue;
                }
                if (_char == "]" && _in_tag) {
                    _in_tag = false;
                    if (_tag == "noeffect") _effect = { type: "none", color: c_white };
                    else if (string_pos("shake", _tag) == 1) _effect.type = string_pos("/shake", _tag) == 0 ? "shake" : "none";
                    else if (string_pos("bounce", _tag) == 1) _effect.type = string_pos("/bounce", _tag) == 0 ? "bounce" : "none";
                    else if (string_pos("pulse", _tag) == 1) _effect.type = string_pos("/pulse", _tag) == 0 ? "pulse" : "none";
                    else if (string_pos("wave", _tag) == 1) _effect.type = string_pos("/wave", _tag) == 0 ? "wave" : "none";
                    else if (string_pos("tremor", _tag) == 1) _effect.type = string_pos("/tremor", _tag) == 0 ? "tremor" : "none";
                    else if (string_pos("float", _tag) == 1) _effect.type = string_pos("/float", _tag) == 0 ? "float" : "none";
                    else if (string_pos("zoom", _tag) == 1) _effect.type = string_pos("/zoom", _tag) == 0 ? "zoom" : "none";
                    else if (string_pos("color=", _tag) == 1) {
                        var _color = string_replace(_tag, "color=", "");
                        if (_color == "red") _effect.color = c_red;
                        else if (_color == "blue") _effect.color = c_blue;
                        else if (_color == "yellow") _effect.color = c_yellow;
                        else if (_color == "purple") _effect.color = c_purple;
                        else if (_color == "orange") _effect.color = c_orange;
                        else if (_color == "green") _effect.color = c_green;
                        else if (_color == "/color") _effect.color = c_white;
                    }
                    continue;
                }
                if (_in_tag) {
                    _tag += _char;
                    continue;
                }
                if (_char == " " || i == string_length(_text)) {
                    if (i == string_length(_text)) _current += _char;
                    if (_current != "") {
                        array_push(words, _current);
                        array_push(effects, { type: _effect.type, color: _effect.color });
                        _current = "";
                    }
                    if (_char == " ") {
                        array_push(words, " ");
                        array_push(effects, { type: "none", color: c_white });
                    }
                } else _current += _char;
            }
        }
    } else {
        if (option_count != 0) draw_options = true;
        else if (keyboard_check_pressed(vk_space) || gamepad_button_check_pressed(global.controller, gp_face1)) {
            instance_destroy();
            global.is_game_paused = false;
            global.dialogue_active = !global.dialogue_active;
        }
    }
}
```

### Evento Alarm\[0\]

Controla a revelação gradual do texto com sons (substitua `snd_dialogue_default` pelo seu som).

```gml
if (is_initialized) {
    global.is_game_paused = true;
    global.dialogue_active = true;
    if (current_char < string_length(dialogue_grid[# DialogueInfo.Text, current_page])) {
        var _char = string_char_at(dialogue_grid[# DialogueInfo.Text, current_page], current_char + 1);
        var _is_tremor = false;
        
        // Verifica se o caractere atual está em uma palavra com [tremor]
        var _current_text = string_copy(dialogue_grid[# DialogueInfo.Text, current_page], 1, current_char + 1);
        for (var i = 0; i < array_length(words); i++) {
            if (string_pos(words[i], _current_text) > 0 && effects[i].type == "tremor") {
                _is_tremor = true;
                break;
            }
        }
        
        if (_char != "." && _char != "," && _char != "-") {
            var _snd = _is_tremor ? snd_dialogue_tremor : snd_dialogue_default;
            audio_play_sound(_snd, 1, 0);
            current_char++;
            alarm[0] = 1;
        } else {
            alarm[0] = 15;
            var _snd = _is_tremor ? snd_dialogue_tremor : snd_dialogue_default;
            audio_play_sound(_snd, 1, 0);
            current_char++;
        }
    }
}
```

### Evento Draw GUI

Desenha a interface do diálogo, incluindo texto, retratos e opções (substitua `fnt_dialogue`, `fnt_name`, `spr_option_bg`, `spr_option_selector` pelos seus assets).

```gml
if (is_initialized) {
    var _gui_width = display_get_gui_width();
    var _gui_height = display_get_gui_height();
    var _x = 0;
    var _y = _gui_height - 200;
    var _c = c_black;
    var _portrait_sprite = dialogue_grid[# DialogueInfo.Portrait, current_page];
    var _text = string_copy(dialogue_grid[# DialogueInfo.Text, current_page], 1, current_char);
    draw_set_font(fnt_dialogue);

    if (dialogue_grid[# DialogueInfo.Side, current_page] == 0) {
        // Caixa à direita, retrato à esquerda
        draw_rectangle_color(_x + 200, _y, _gui_width, _gui_height, _c, _c, _c, _c, false);
        draw_rectangle_color(_x + 200, _y + 1, _gui_width, _gui_height, c_white, c_white, c_white, c_white, true);
        draw_rectangle_color(_x + 201, _y + 2, _gui_width, _gui_height, c_white, c_white, c_white, c_white, true);
        draw_rectangle_color(_x + 202, _y + 3, _gui_width, _gui_height, c_white, c_white, c_white, c_white, true);
        draw_text_effect(_x + 264, _y + 32, _text, 32, _gui_width - 264, words, effects);
        draw_set_font(fnt_name);
        draw_set_color(c_white);
        draw_text(_x + 216, _y - 32, dialogue_grid[# DialogueInfo.Name, current_page]);
        draw_set_font(-1);
        draw_set_color(c_white);
        draw_sprite_ext(_portrait_sprite, 0, 100, _gui_height, 3, 3, 0, c_white, 1);
    } else {
        // Caixa à esquerda, retrato à direita
        draw_rectangle_color(_x, _y, _gui_width - 200, _gui_height, _c, _c, _c, _c, false);
        draw_rectangle_color(_x, _y + 1, _gui_width - 200, _gui_height, c_white, c_white, c_white, c_white, true);
        draw_rectangle_color(_x, _y + 2, _gui_width - 201, _gui_height, c_white, c_white, c_white, c_white, true);
        draw_rectangle_color(_x, _y + 3, _gui_width - 202, _gui_height, c_white, c_white, c_white, c_white, true);
        draw_text_effect(_x + 64, _y + 32, _text, 32, _gui_width - 264, words, effects);
        var _strw = string_width(dialogue_grid[# DialogueInfo.Name, current_page]);
        draw_set_font(fnt_name);
        draw_set_color(c_white);
        draw_text(_gui_width - 216 - _strw, _y - 32, dialogue_grid[# DialogueInfo.Name, current_page]);
        draw_set_font(-1);
        draw_set_color(c_white);
        draw_sprite_ext(_portrait_sprite, 0, _gui_width - 100, _gui_height, -3, 3, 0, c_white, 1);
    }

    draw_set_font(fnt_dialogue);
    if (draw_options) {
        var _opx = _x + 32;
        var _opy = _y - 32;
        var _opsep = 48;
        var _opborda = 6;
        selected_option += keyboard_check_pressed(ord("W")) - keyboard_check_pressed(ord("S"));
        selected_option = clamp(selected_option, 0, option_count - 1);

        for (var i = 0; i < option_count; i++) {
            var _stringw = string_width(option[i]);
            draw_sprite_ext(spr_option_bg, 0, _opx, _opy - (_opsep * i), (_stringw + _opborda * 2) / 16, 1, 0, c_white, 1);
            draw_text(_opx + _opborda, _opy - (_opsep * i), option[i]);
            if (selected_option == i) {
                draw_sprite(spr_option_selector, 0, _x + 8, _opy - (_opsep * i) + 8);
            }
        }

        if (keyboard_check_pressed(vk_enter) || gamepad_button_check_pressed(global.controller, gp_face1)) {
            var _dialogue = instance_create_layer(x, y, "Dialogue", obj_dialogo);
            _dialogue.npc_dialogue_id = option_response[selected_option];
            instance_destroy();
        }
    }
    draw_set_font(-1);
}
```

## Passo 3: Integração com o Jogador

No objeto do jogador (ex.: `obj_player`), adicione no evento **Step** para detectar interação com NPCs.

```gml
if (distance_to_object(obj_par_npc) <= 15) {
    if (keyboard_check_pressed(ord("F")) || gamepad_button_check_pressed(global.controller, gp_face2)) {
        global.dialogue_active = true;
        var _npc = instance_nearest(x, y, obj_par_npc);
        var _dialogue = instance_create_layer(x, y, "Dialogue", obj_dialogo);
        _dialogue.npc_dialogue_id = _npc.dialogue_id;
    }
}
```

## Passo 4: Script de Diálogos (scr_dialogues)

Este script centraliza todos os diálogos em um switch baseado em `npc_dialogue_id`. Crie scripts adicionais para outros idiomas (ex.: `scr_dialogues_en`).

### Estrutura Geral

```gml
function scr_dialogues() {
    switch (npc_dialogue_id) {
        case "example_dialogue":
            ds_grid_add_text("Hello, welcome to the game!", spr_portrait_default, 0, "NPC");
            ds_grid_add_text("This is a test dialogue.", spr_portrait_default, 1, "NPC");
            break;
        
        case "quest_dialogue":
            ds_grid_add_text("Would you like to start a quest?", spr_portrait_default, 1, "NPC");
            add_option("Yes, let's do it!", "response_yes");
            add_option("No, maybe later.", "response_no");
            break;
        
        case "response_yes":
            // Exemplo: Ativar uma missão
            // global.quests[0].status = "active";
            ds_grid_add_text("Great, let's begin!", spr_portrait_default, 0, "NPC");
            break;
        
        case "response_no":
            ds_grid_add_text("Alright, come back later.", spr_portrait_default, 0, "NPC");
            break;
    }
}
```

### Funções Auxiliares

Adicione ao final do script para gerenciar a grid, texto e opções.

```gml
function ds_grid_add_row(_grid) {
    ds_grid_resize(_grid, ds_grid_width(_grid), ds_grid_height(_grid) + 1);
    return (ds_grid_height(_grid) - 1);
}

function ds_grid_add_text(_text, _portrait, _side, _name) {
    var _grid = dialogue_grid;
    var _y = ds_grid_add_row(_grid);
    
    _grid[# DialogueInfo.Text, _y] = _text;
    _grid[# DialogueInfo.Portrait, _y] = _portrait;
    _grid[# DialogueInfo.Side, _y] = _side;
    _grid[# DialogueInfo.Name, _y] = _name;
}

function add_option(_option_text, _response_id) {
    option[option_count] = _option_text;
    option_response[option_count] = _response_id;
    option_count++;
}

function draw_text_effect(_x, _y, _text, _sep, _width, _words, _effects) {
    var _current_x = _x;
    var _current_y = _y;
    var _char_count = 0;
    
    for (var i = 0; i < array_length(_words); i++) {
        var _word = _words[i];
        var _word_len = string_length(_word);
        var _word_width = string_width(_word);
        
        // Verifica se a palavra cabe na linha
        if (_current_x + _word_width > _x + _width) {
            _current_y += _sep;
            _current_x = _x;
        }
        
        // Aplica efeito por palavra
        var _effect_x = _current_x;
        var _effect_y = _current_y;
        var _effect_color = c_white;
        var _effect_scale = 1;
        
        if (i < array_length(_effects)) {
            var _effect = _effects[i];
            if (_effect.type == "shake") {
                _effect_x += random_range(-1.5, 2);
                _effect_y += random_range(-1.5, 2);
            }
            if (_effect.type == "bounce") {
                _effect_y += sin(current_time * 0.005 + i) * 4;
            }
            if (_effect.type == "pulse") {
                _effect_scale = 1 + sin(current_time * 0.005 + i) * 0.15;
            }
            if (_effect.type == "zoom") {
                _effect_scale = 1 + sin(current_time * 0.01 + i) * 0.2;
            }
            if (_effect.type == "float") {
                _effect_y += sin(current_time * 0.003 + i) * 1;
            }
            if (_effect.type == "wave" || _effect.type == "tremor") {
                // Aplicado por caractere no loop abaixo
            }
            _effect_color = _effect.color;
        }
        
        // Desenha caractere por caractere
        for (var j = 1; j <= _word_len; j++) {
            _char_count++;
            if (_char_count > string_length(_text)) break;
            
            var _char = string_char_at(_word, j);
            var _char_width = string_width(_char);
            
            // Aplica efeitos por caractere
            var _char_x = _effect_x;
            var _char_y = _effect_y;
            if (i < array_length(_effects)) {
                if (_effects[i].type == "wave") {
                    _char_y += sin(current_time * 0.003 + (i * 0.5 + j * 0.2)) * 4;
                }
                if (_effects[i].type == "tremor") {
                    _char_x += random_range(-1.8, 2.5);
                    _char_y += random_range(-1.8, 2.5);
                }
            }
            
            draw_set_color(_effect_color);
            draw_text_transformed(_char_x, _char_y, _char, _effect_scale, _effect_scale, 0);
            
            _effect_x += _char_width * _effect_scale;
        }
        
        _current_x += _word_width;
        if (_char_count > string_length(_text)) break;
    }
    
    draw_set_color(c_white);
}
```

## Como Adicionar Novos Diálogos

1. Crie um NPC filho de `obj_par_npc` e defina `dialogue_id = "new_dialogue";` no Create.
2. No `scr_dialogues`, adicione:

   ```gml
   case "new_dialogue":
       ds_grid_add_text("Your text here", spr_your_portrait, 0, "Character Name");
       // Adicione mais linhas ou opções
       break;
   ```
3. Para opções, use:

   ```gml
   ds_grid_add_text("Would you like to proceed?", spr_your_portrait, 1, "Character Name");
   add_option("Yes", "response_yes");
   add_option("No", "response_no");
   ```
4. Crie cases para respostas:

   ```gml
   case "response_yes":
       ds_grid_add_text("Let's go!", spr_your_portrait, 0, "Character Name");
       break;
   ```

## Efeitos no Texto

Use tags para aplicar efeitos visuais:

- `[shake]` / `[/shake]`: Tremor leve na palavra.
- `[bounce]` / `[/bounce]`: Salto vertical.
- `[pulse]` / `[/pulse]`: Pulsação de escala.
- `[wave]` / `[/wave]`: Onda por caractere.
- `[tremor]` / `[/tremor]`: Tremor forte por caractere.
- `[float]` / `[/float]`: Flutuação suave.
- `[zoom]` / `[/zoom]`: Zoom in/out.
- `[color=red]` / `[/color]`: Muda cor (red, blue, yellow, purple, orange, green).
- `[noeffect]`: Reseta todos os efeitos.

Exemplo: `"Welcome [shake]to[/shake] the [color=red]game[/color]!"`

## Configuração de Assets

Substitua os seguintes placeholders pelos seus assets no editor do GMS2:

- **Sprites**:
  - `spr_portrait_default`: Retrato padrão para NPCs.
  - `spr_option_bg`: Fundo das opções de escolha.
  - `spr_option_selector`: Indicador da opção selecionada.
- **Sons**:
  - `snd_dialogue_default`: Som padrão para cada caractere.
  - `snd_dialogue_tremor`: Som para efeito `[tremor]`.
- **Fontes**:
  - `fnt_dialogue`: Fonte para o texto do diálogo.
  - `fnt_name`: Fonte para o nome do personagem.

## Caso você queira contribuir com o meu trabalho como desenvolvedor desse sistema você pode me apoiar no meu jogo Geocraft, acesse geocomunity geocraft no youtube!!!