-- ===================================
-- ⚡ CARREGAMENTO ULTRA-RÁPIDO FLASH
-- ===================================

local VIDA_ADDR, COLETE_ADDR, PLAYER_BASE, VEHICLE_SPEED_ADDRS, MAP_BASE, PLAYER_FLAGS

function carregarFlash()
    gg.toast("⚡ Carregando ultra-rápido...")

    -- ❤️ VIDA → intervalo O, 36 resultados
    if not VIDA_ADDR then
        gg.clearResults()
        gg.setRanges(gg.REGION_OTHER)
        gg.searchNumber("125.18000030518", gg.TYPE_FLOAT)
        VIDA_ADDR = gg.getResults(36)
        gg.clearResults()
    end

    -- 🛡️ COLETE → intervalo O, 36 resultados
    if not COLETE_ADDR then
        gg.clearResults()
        gg.setRanges(gg.REGION_OTHER)
        gg.searchNumber("125.18000030518", gg.TYPE_FLOAT)
        COLETE_ADDR = gg.getResults(36)
        gg.clearResults()
    end

    -- 👤 PERSONAGEM (TP) → intervalo O, 1 resultado
    if not PLAYER_BASE then
        gg.clearResults()
        gg.setRanges(gg.REGION_OTHER)
        gg.searchNumber("999.765625", gg.TYPE_FLOAT)
        local r = gg.getResults(1)
        if #r > 0 then
            PLAYER_BASE = r[1].address
            PLAYER_FLAGS = r[1].flags
        end
        gg.clearResults()
    end

    -- 🚗 SPEED VEÍCULO → intervalo Cb + O, 2 resultados
    if not VEHICLE_SPEED_ADDRS then
        gg.clearResults()
        gg.setRanges(gg.REGION_C_BSS | gg.REGION_OTHER)
        gg.searchNumber("479320161982472000000000", gg.TYPE_FLOAT)
        local r = gg.getResults(2)
        if #r > 0 then
            VEHICLE_SPEED_ADDRS = { r[1].address + 64, r[2].address + 64 }
        end
        gg.clearResults()
    end

    -- 🗺️ TP MAPA → intervalo Cb, 1 resultado
    if not MAP_BASE then
        gg.clearResults()
        gg.setRanges(gg.REGION_C_BSS)
        gg.searchNumber(tostring(0x675F726164617212), gg.TYPE_QWORD)
        local r = gg.getResults(1)
        if #r > 0 then MAP_BASE = r[1].address end
        gg.clearResults()
    end

    gg.toast("✅ Carregamento FLASH concluído!")
end

-- CHAMADA ÚNICA
carregarFlash()


local EXPIRY = 999999999999999
local now = os.time()
if now >= EXPIRY then
  if gg and gg.alert then gg.alert("⚠️ SCRIPT EXPIRADO — contate o criador") end
  return
else
  if gg and gg.toast then gg.toast("⏳ Script válido por "..(EXPIRY - now).." segundos") end
end
local GLOBAL_EXIT_SIGNAL = 'EXIT_ALL' 
local MAP_TP_LOOP_ACTIVE = false 
local LAST_TP_X, LAST_TP_Y, LAST_TP_Z = nil, nil, nil 
local LAST_TP_TIME = os.time() * 1000 
local base = nil 
local flags = nil 
function findBase()
 gg.clearResults()
 gg.setRanges(gg.REGION_OTHER) -- << REGIÃO DO PERSONAGEM
 gg.searchNumber('999.765625', gg.TYPE_FLOAT)
 local r = gg.getResults(1)
 if #r > 0 then
   base = r[1].address
   flags = r[1].flags
   gg.clearResults()
   gg.toast('✅ Base de Teleporte encontrada!')
   return true
 else
   gg.toast('❌ Endereco base do personagem nao encontrado.')
   return false
 end
end
function checkBaseLoaded()
  if base == nil then 
     return findBase() -- findBase retorna true ou false
  end
  return true
end
function tp(x,y,z)
 if not checkBaseLoaded() then 
    gg.toast('⚠️ Endereco de teleporte nao carregado. Fique parado e re-execute o script.')
    return 
 end
 gg.setValues({
  {address = base + 96, flags = flags, value = x},
  {address = base + 100, flags = flags, value = y},
  {address = base + 104, flags = flags, value = z}
 })
end
local _mapCore = {anchor = nil}
local MAP_COORD_X_OFFSET = 0x78
local MAP_COORD_Z_OFFSET = 0x7C
local MAP_COORD_Y_OFFSET = 0x80
local ALTITUDE_ADJUSTMENT = 2.0 
function encontrarBaseMapa()
  if _mapCore.anchor then return _mapCore.anchor end
  gg.clearResults()
  gg.toast("🔍 Procurando valor fixo do mapa...")
  local v = tonumber("0x675F726164617212") 
  gg.setRanges(gg.REGION_C_BSS) -- << REGIÃO DO MAPA
  gg.searchNumber(tostring(v), gg.TYPE_QWORD)
  local results = gg.getResults(1)
  if #results == 0 then
    return nil
  end
  _mapCore.anchor = results[1].address
  gg.toast("✅ Base do mapa encontrada!")
  return _mapCore.anchor
end
function lerCoordenadasMapa()
  local baseMapa = _mapCore.anchor 
  if not baseMapa then return nil, nil, nil end
  local vals = gg.getValues({
    {address = baseMapa + MAP_COORD_X_OFFSET, flags = gg.TYPE_FLOAT},
    {address = baseMapa + MAP_COORD_Z_OFFSET, flags = gg.TYPE_FLOAT}, 
    {address = baseMapa + MAP_COORD_Y_OFFSET, flags = gg.TYPE_FLOAT} 
  })
  local x, z_mapa, y_mapa = vals[1].value, vals[2].value, vals[3].value
  if x == 0 and z_mapa == 0 and y_mapa == 0 then return nil, nil, nil end
  local final_x = x
  local final_y = z_mapa
  local final_z = y_mapa + ALTITUDE_ADJUSTMENT  
  return final_x, final_y, final_z 
end
function executeMapTpOnce()
    local x, y, z = lerCoordenadasMapa()
    if x and y and z then
        local isNewPoint = (math.abs(x - (LAST_TP_X or 0.0)) > 0.5) or 
                           (math.abs(y - (LAST_TP_Y or 0.0)) > 0.5) or
                           (math.abs(z - (LAST_TP_Z or 0.0)) > 0.5)
        local timeSinceLastTP = (os.time() * 1000) - LAST_TP_TIME 
        local shouldTP = timeSinceLastTP >= 500
        if isNewPoint and shouldTP then
            tp(x, y, z)
            LAST_TP_X, LAST_TP_Y, LAST_TP_Z = x, y, z
            LAST_TP_TIME = os.time() * 1000 
            gg.toast(string.format("✅ TP Ativo para (X:%.1f, Y:%.1f, Z:%.1f)", x, y, z))
        end
    end
end
function startMapTpLoop()
    if not checkBaseLoaded() then 
       gg.alert("❌ Base do personagem não carregada. Re-execute o script.") 
       return 
    end
    if not encontrarBaseMapa() then 
       gg.alert("❌ Base do mapa não encontrada.") 
       return 
    end
    MAP_TP_LOOP_ACTIVE = true 
    gg.toast("✅ auto Got Map atv")
    LAST_TP_X, LAST_TP_Y, LAST_TP_Z = nil, nil, nil
    LAST_TP_TIME = os.time() * 1000 
end
function stopMapTpLoop()
    MAP_TP_LOOP_ACTIVE = false
    LAST_TP_X, LAST_TP_Y, LAST_TP_Z = nil, nil, nil
    gg.toast("🛑 auto Got Map Dstv")
end
function rotinaTeleportarManual()
  if not checkBaseLoaded() then 
     gg.alert("❌ Base do personagem não carregada. Re-execute o script.") 
     return 
  end
  if not encontrarBaseMapa() then 
     gg.alert("❌ Base do mapa não encontrada.") 
     return 
  end
  gg.toast("⌛ Aguarde até marcar um ponto no mapa...")
  local x, y, z
  local max_wait_time = 15000 
  local start_time = os.time()
  while os.time() - start_time < max_wait_time do
    x, y, z = lerCoordenadasMapa()
    if x and y and z then break end
    gg.sleep(200)
    if not gg.isVisible(true) and gg.choice({"Continuar esperando?"}, nil, "Mapa não marcado. Cancelar?") == nil then
        gg.toast("❌ Teleporte cancelado pelo usuário.")
        return 
    end
  end
  if not (x and y and z) then
     gg.alert("❌ Nenhuma marcação válida no mapa detectada após " .. max_wait_time/1000 .. " segundos.")
     return
  end
  tp(x, y, z)
  gg.toast(string.format("✅ Teleportado para o ponto do mapa (X:%.1f, Y:%.1f, Z:%.1f)", x, y, z))
end
local farmCoords = 
{
 {x=808.777, y=23.405, z=1009.424, delay=26000},
 {x=834.226, y=21.318, z=1012.063, delay=100},
 {x=831.588, y=43.862, z=1014.801, delay=26000},
 {x=834.226, y=21.318, z=1012.063, delay=100},
 {x=858.220, y=19.414, z=1012.000, delay=26000},
 {x=834.226, y=21.318, z=1012.063, delay=100}
}
local minaCoord = {x=-90.538, y=-1574.110, z=63.847}
local farmLoop = false
local ifoodEmpregoCoord = {x=-1637.465, y=-1271.093, z=68.399} 
local prisaoExitCoord = {x=100.0, y=-2000.0, z=68.0} 
local speedCache = { addresses = nil, lastTarget = '3.14159274101', lastSpeed = nil }
local speedAgachadoCache = { addresses = nil, lastTarget = '974.40002441406', lastSpeed = nil }
local superPuloCache = { addresses = nil, lastTarget = '4.15768349e21', lastJumpValue = nil }
local coleteCache = { endereco = nil, valorOriginal = 125.18000030518, ultimoValor = nil } 
local vidaCache = { endereco = nil, valorOriginal = 125.18000030518, ultimoValor = nil, offset = -4 }
local lightMapCache = { addresses = nil, lastTarget = '974.40002441406', lastValue = nil }
local vegetacaoCache = { addresses = nil, lastTarget = '223.27423095703', lastValue = nil }
local vehicleSpeedCache = { addresses = nil, lastSpeedValue = nil, lastTargetValue = nil }
function startFarmMina()
 if not checkBaseLoaded() then gg.toast('⚠️ Endereco de teleporte nao carregado. Fique parado e re-execute o script.');
 return end
 farmLoop = true
 gg.toast('⛏️ Farm Mina iniciado! Pressione o icone do GG para PARAR.')
 while farmLoop do
   for _,p in ipairs(farmCoords) do
     if not farmLoop then break end 
     tp(p.x, p.y, p.z)
     gg.sleep(p.delay)
   end
 end
 gg.toast('🛑 Farm Mina parado.')
end
function teleportMina()
 if not checkBaseLoaded() then gg.toast('⚠️ Endereco de teleporte nao carregado. Fique parado e re-execute o script.');
 return end
 tp(minaCoord.x, minaCoord.y, minaCoord.z)
 gg.toast('📍 Teleportado para Mina.')
end
function teleportIfoodEmprego()
    if not checkBaseLoaded() then 
        gg.toast('⚠️ Endereco de teleporte nao carregado. Fique parado e re-execute o script.');
        return 
    end
    tp(ifoodEmpregoCoord.x, ifoodEmpregoCoord.y, ifoodEmpregoCoord.z)
    gg.toast('📍 Teleportado para Emprego iFood.')
end
function sairDaPrisao()
    if not checkBaseLoaded() then 
        gg.toast('⚠️ Endereco de teleporte nao carregado. Fique parado e re-execute o script.');
        return 
    end
    tp(prisaoExitCoord.x, prisaoExitCoord.y, prisaoExitCoord.z)
    gg.toast('🚶‍♂️ Teleportado para fora da prisão.')
end
local ACTIVE = false 
local STEP = 0
local RETURN_POS = {x=nil, y=nil, z=nil}
local X_OFFSET = 96
local Y_OFFSET = 100
local Z_OFFSET = 104
local RADIUS = 7
local POINTS = {
    [1] = {x=-1649.9599609375, y=-1243.699951170, z=68.3899841308},
    [2] = {x=1593.62255859375, y=-898.7886962890625, z=2067.13720703125},
    [3] = {x=1595.196044921875, y=-888.9354858398438, z=2067.197998046875},
    [4] = {x=1593.62255859375, y=-898.7887962890625, z=2067.13720703125},
    [5] = {x=-1649.9599609375, y=-1243.699951170, z=68.3999841308}
}
function readPlayer()
    if not base then return 0,0,0 end
    local v = gg.getValues({
        {address = base + X_OFFSET, flags = flags},
        {address = base + Y_OFFSET, flags = flags},
        {address = base + Z_OFFSET, flags = flags}
    })
    return tonumber(v[1].value), tonumber(v[2].value), tonumber(v[3].value) 
end
function dist(x1,y1,z1,x2,y2,z2)
    local dx=x1-x2
    local dy=y1-y2
    local dz=z1-z2
    return math.sqrt(dx*dx + dy*dy + dz*dz)
end
function saveReturn()
    if base == nil then
        return false 
    end
    local x,y,z = readPlayer()
    RETURN_POS.x = x
    RETURN_POS.y = y
    RETURN_POS.z = z
    return true 
end
function returnBack()
    if base and RETURN_POS.x then
        gg.setValues({
            {address = base + X_OFFSET, flags = flags, value = RETURN_POS.x},
            {address = base + Y_OFFSET, flags = flags, value = RETURN_POS.y},
            {address = base + Z_OFFSET, flags = flags, value = RETURN_POS.z}
        })
        gg.toast("🔙 Retorno final.")
    end
    ACTIVE = false
    STEP = 0
    RETURN_POS.x = nil 
end
function runStep()
    if not ACTIVE or not base then return end
    local px,py,pz = readPlayer()
    if STEP == 1 then
        local t = POINTS[2]
        if dist(px,py,pz,t.x,t.y,t.z) <= RADIUS then
            gg.setValues({
                {address = base + X_OFFSET, flags = flags, value = POINTS[3].x},
                {address = base + Y_OFFSET, flags = flags, value = POINTS[3].y},
                {address = base + Z_OFFSET, flags = flags, value = POINTS[3].z}
            })
            gg.toast("✅ P2 detectado → TP P3")
            gg.sleep(2500)
            gg.setValues({
                {address = base + X_OFFSET, flags = flags, value = POINTS[4].x},
                {address = base + Y_OFFSET, flags = flags, value = POINTS[4].y},
                {address = base + Z_OFFSET, flags = flags, value = POINTS[4].z}
            })
            gg.sleep(500) 
            gg.toast("✅ TP para P4 → Vá até P5 (Você caminha daqui)")
            STEP = 2
        end
    elseif STEP == 2 then
        local t = POINTS[5]
        if dist(px,py,pz,t.x,t.y,t.z) <= RADIUS then
            gg.toast("✅ P5 detectado → Retorno")
            returnBack() 
        end
    end
end
function toggleFarm()
    if not checkBaseLoaded() then 
        gg.toast('⚠️ Endereco de teleporte nao carregado. Fique parado e re-execute o script.');
        return 
    end
    if farmLoop then 
        farmLoop = false
        gg.toast("🛑 Farm Mina DESLIGADO.")
    end
    if ACTIVE then
        returnBack() 
        return
    end
    if not saveReturn() then 
        gg.toast("❌ Não foi possível salvar a posição de retorno.")
        return
    end
    gg.setValues({
        {address = base + X_OFFSET, flags = flags, value = POINTS[1].x},
        {address = base + Y_OFFSET, flags = flags, value = POINTS[1].y},
        {address = base + Z_OFFSET, flags = flags, value = POINTS[1].z}
    })
    ACTIVE = true
    STEP = 1
    gg.toast("✅ Farm iFood (5 Pts) LIGADO → Vá para P2. Pressione o GG para desligar/retornar.")
end
function menuFarm()
    while true do 
        local farmMenu = gg.choice({
            '-----[👷 Farm Mina]------',            
            '⛏ Farm mina (Loop)',                       
            '⛏ Ir Emprego Mina',                  
            '-----[👨‍🍳 Farm iFood]------',      
            '🥓 Farm ifood v2.0',  
            '🥓 Ir Emprego iFood', 
            '◀️ VOLTAR PARA MENU PRINCIPAL'            
        }, nil, 'MENU FARM SCRIPT DSL ')
        if not farmMenu then return GLOBAL_EXIT_SIGNAL end 
        if farmMenu == 7 then return end   
        if farmMenu == 2 then 
            try_call_with_exit(startFarmMina, 'fm_start')
        elseif farmMenu == 3 then 
            try_call_with_exit(teleportMina, 'fm_tp') 
        elseif farmMenu == 5 then          
            toggleFarm()
            return 
        elseif farmMenu == 6 then 
            try_call_with_exit(teleportIfoodEmprego, 'ifood_tp_emprego')
        end
    end
end
function try_call_with_exit(fn, tag)
   local ok, result = pcall(fn)
   if not ok then
     gg.toast('❌ Erro no Menu ['..tostring(tag)..']: '..tostring(result))
     return nil
   end
   return result
end
function verificarAutenticacao() end
function verificarExpiracao() end

function setFloat(address, value, message)
   gg.setValues({{address = address, flags = gg.TYPE_FLOAT, value = value}})
   if message then gg.toast(message) end
end
function ativarspeed(velocidadespeed) 
    if not speedCache.addresses then
        gg.clearResults();
        gg.setRanges(gg.REGION_OTHER) -- << REGIÃO DE PESQUISA (PERSONAGEM)
        gg.searchNumber(speedCache.lastTarget, gg.TYPE_FLOAT)
        local results = gg.getResults(500) 
        if #results == 0 then gg.toast('❌ Endereços de Speed não encontrados.'); return false end
        speedCache.addresses = {}
        for i, result in ipairs(results) do speedCache.addresses[i] = result.address - 36 end
    end
    
    local edits = {}
    for i, addr in ipairs(speedCache.addresses) do edits[i] = { address = addr, flags = gg.TYPE_FLOAT, value = velocidadespeed } end
    gg.setValues(edits)
    speedCache.lastSpeed = velocidadespeed
    gg.toast(string.format('🏃 Speed: %.1fx', velocidadespeed))
    gg.clearResults()
    return true
end
function speedHackOptions()
   while true do
     local speedMenu = gg.choice({
           '>> 🏃 Speed Hack 2x', '>> 🏃 Speed Hack 4x', '>> 🏃 Speed Hack 6x', '>> 🏃 Speed Hack 8x', '>>🚶 Anda Normal (1x)', '◀️ Voltar'
       }, nil, 'MENU SPEED HACK:')
       if not speedMenu then return GLOBAL_EXIT_SIGNAL end 
       if speedMenu == 6 then return end 
       if speedMenu == 1 then ativarspeed(2)
       elseif speedMenu == 2 then ativarspeed(4)
       elseif speedMenu == 3 then ativarspeed(6)
       elseif speedMenu == 4 then ativarspeed(8)
       elseif speedMenu == 5 then ativarspeed(1) end
   end
end
function ativarspeedagachado(velocidadespeed) 
    if not speedAgachadoCache.addresses then
        gg.clearResults(); 
        gg.setRanges(gg.REGION_OTHER) -- << REGIÃO DE PESQUISA (PERSONAGEM)
        gg.searchNumber(speedAgachadoCache.lastTarget, gg.TYPE_FLOAT)
        local results = gg.getResults(500) 
        if #results == 0 then gg.toast('❌ Endereços de Speed Agachado não encontrados.'); return false end
        speedAgachadoCache.addresses = {}
        for i, result in ipairs(results) do speedAgachadoCache.addresses[i] = result.address - 72 end
    end
    
    local edits = {}
    for i, addr in ipairs(speedAgachadoCache.addresses) do edits[i] = { address = addr, flags = gg.TYPE_FLOAT, value = velocidadespeed } end
    gg.setValues(edits)
    speedAgachadoCache.lastSpeed = velocidadespeed
    gg.toast(string.format('🐢 Speed Agachado: %.1fx', velocidadespeed))
    gg.clearResults()
    return true
end
function speedHackOptionsagachado()
   while true do
      local speedMenuagachado = gg.choice({
           '🧎 Speed Hack Agachado 2x', '🧎 Speed Hack Agachado 4x', '🧎 Speed Hack Agachado 6x', '🧎 Speed Hack Agachado 8x', '◀️ VOLTAR'
       }, nil, 'MENU SPEED HACK AGACHADO:')
       if not speedMenuagachado then return GLOBAL_EXIT_SIGNAL end 
       if speedMenuagachado == 5 then return end    
       if speedMenuagachado == 1 then ativarspeedagachado(2)
       elseif speedMenuagachado == 2 then ativarspeedagachado(4)
       elseif speedMenuagachado == 3 then ativarspeedagachado(6)
       elseif speedMenuagachado == 4 then ativarspeedagachado(8) end
   end
end
function ativarSuperPulo2(valordosuperpulo) 
    if not superPuloCache.addresses then 
        gg.clearResults(); 
        gg.setRanges(gg.REGION_OTHER) -- << REGIÃO DE PESQUISA (PERSONAGEM)
        gg.searchNumber(superPuloCache.lastTarget, gg.TYPE_FLOAT)
        local results = gg.getResults(500) 
        if #results == 0 then gg.toast('❌ Endereços de Super Pulo não encontrados.'); return false end
        superPuloCache.addresses = {}
        -- Usando offset padrão para Super Pulo: -164
        for i, result in ipairs(results) do superPuloCache.addresses[i] = result.address - 164 end 
    end
    
    local edits = {}
    for i, addr in ipairs(superPuloCache.addresses) do edits[i] = { address = addr, flags = gg.TYPE_FLOAT, value = valordosuperpulo } end
    gg.setValues(edits)
    superPuloCache.lastJumpValue = valordosuperpulo
    gg.toast(string.format('⬆️ Super Pulo: %.1f', valordosuperpulo))
    gg.clearResults()
    return true
end
function desativarSuperPulo2()
   local defaultJumpValue = 0.8
   if not superPuloCache.addresses then 
       -- Se não tem o endereço, tenta buscar primeiro antes de desativar (busca no momento)
       ativarSuperPulo2(defaultJumpValue)
       return true
   end
   local edits = {}
   for i, addr in ipairs(superPuloCache.addresses) do edits[i] = { address = addr, flags = gg.TYPE_FLOAT, value = defaultJumpValue } end 
   gg.setValues(edits)
   superPuloCache.lastJumpValue = defaultJumpValue
   gg.toast('⬇️ Super Pulo DESATIVADO')
   gg.clearResults()
   return true
end
function menuSuperPulo2()
   while true do
     local menuSuperPulo = gg.choice({
         '>> 🦘 SUPER PULO 2X (2.56)', '>> 🦘 SUPER PULO 4X (4.56)', '>> 🦘 SUPER PULO 6X (6.56)', '>> 🦘 SUPER PULO 8X (8.56)', '>> 🦘 Desativar Super Pulo', '◀️ Voltar'
       }, nil, 'MENU SUPER PULO')
       if not menuSuperPulo then return GLOBAL_EXIT_SIGNAL end 
       if menuSuperPulo == 6 then return end  
       if menuSuperPulo == 1 then ativarSuperPulo2(2.56)
       elseif menuSuperPulo == 2 then ativarSuperPulo2(4.56)
       elseif menuSuperPulo == 3 then ativarSuperPulo2(6.56)
       elseif menuSuperPulo == 4 then ativarSuperPulo2(8.56)
       elseif menuSuperPulo == 5 then desativarSuperPulo2() end
   end
end
function alterarColete(novoValor) 
    if not coleteCache.endereco then 
        gg.clearResults(); 
        gg.setRanges(gg.REGION_OTHER) -- << REGIÃO DE PESQUI
