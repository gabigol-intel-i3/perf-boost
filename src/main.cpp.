#include <Geode/Geode.hpp>
#include <Geode/modify/MenuLayer.hpp>
#include <Geode/modify/GJBaseGameLayer.hpp>
#include <algorithm>
#include <chrono>
#include <vector>

using namespace geode::prelude;

// ---------- Configuración ----------

struct PerfCfg {
	float particles; // 0..1
	bool noShake;
};

static PerfCfg getCfg() {
	auto p = Mod::get()->getSettingValue<std::string>("preset");
	if (p == "off") return {1.f, false};
	if (p == "balanced") return {0.5f, false};
	if (p == "aggressive") return {0.25f, true};
	if (p == "extreme") return {0.f, true};
	// custom
	return {
		static_cast<float>(Mod::get()->getSettingValue<int64_t>("custom-particles")) / 100.f,
		Mod::get()->getSettingValue<bool>("custom-no-shake")
	};
}

static int64_t s_appliedFps = -1;

static void applyFps() {
	if (!Mod::get()->getSettingValue<bool>("apply-fps")) return;
	auto fps = Mod::get()->getSettingValue<int64_t>("target-fps");
	if (fps == s_appliedFps || fps <= 0) return;
	s_appliedFps = fps;
	CCApplication::sharedApplication()->setAnimationInterval(1.0 / static_cast<double>(fps));
}

// ---------- Contador de FPS / frametime ----------

class PerfOverlay : public CCNode {
protected:
	CCLabelBMFont* m_label = nullptr;
	std::chrono::steady_clock::time_point m_last;
	std::vector<float> m_ring;
	size_t m_idx = 0;
	size_t m_count = 0;
	float m_acc = 0.f;
	int m_frames = 0;

	bool init() override {
		if (!CCNode::init()) return false;

		m_ring.assign(300, 16.7f);
		m_label = CCLabelBMFont::create("", "chatFont.fnt");
		this->addChild(m_label);
		m_last = std::chrono::steady_clock::now();

		this->refresh("");
		this->scheduleUpdate();
		return true;
	}

	void refresh(std::string const& text) {
		auto mod = Mod::get();
		auto win = CCDirector::get()->getWinSize();
		auto corner = mod->getSettingValue<std::string>("counter-corner");
		float const m = 6.f;

		bool top = corner.rfind("top", 0) == 0;
		bool left = corner.find("left") != std::string::npos;

		m_label->setAnchorPoint({left ? 0.f : 1.f, top ? 1.f : 0.f});
		m_label->setPosition({left ? m : win.width - m, top ? win.height - m : m});
		m_label->setScale(static_cast<float>(mod->getSettingValue<double>("counter-scale")));
		m_label->setOpacity(static_cast<GLubyte>(mod->getSettingValue<double>("counter-opacity") * 255.0));
		m_label->setVisible(mod->getSettingValue<bool>("show-counter"));
		if (!text.empty()) m_label->setString(text.c_str());
	}

	void update(float) override {
		// Reloj real (no se ve afectado por el time scale del juego)
		auto now = std::chrono::steady_clock::now();
		float ms = std::chrono::duration<float, std::milli>(now - m_last).count();
		m_last = now;

		m_ring[m_idx] = ms;
		m_idx = (m_idx + 1) % m_ring.size();
		m_count = std::min(m_count + 1, m_ring.size());
		m_acc += ms;
		m_frames++;

		if (m_acc < 500.f) return;

		float fps = m_frames * 1000.f / m_acc;
		float avgMs = m_acc / m_frames;
		m_acc = 0.f;
		m_frames = 0;

		auto mod = Mod::get();
		std::string text = fmt::format("{:.0f} FPS", fps);

		if (mod->getSettingValue<bool>("show-frametime")) {
			text += fmt::format("\n{:.2f} ms", avgMs);
		}
		if (mod->getSettingValue<bool>("show-low")) {
			std::vector<float> v(m_ring.begin(), m_ring.begin() + m_count);
			std::sort(v.begin(), v.end(), std::greater<float>());
			size_t k = std::max<size_t>(1, v.size() / 100);
			float worst = 0.f;
			for (size_t i = 0; i < k; i++) worst += v[i];
			worst /= static_cast<float>(k);
			text += fmt::format("\n1% low: {:.0f}", 1000.f / worst);
		}

		this->refresh(text);
		applyFps(); // por si cambió el ajuste
	}

public:
	static PerfOverlay* create() {
		auto ret = new PerfOverlay();
		if (ret && ret->init()) {
			ret->autorelease();
			return ret;
		}
		CC_SAFE_DELETE(ret);
		return nullptr;
	}
};

// ---------- Hooks ----------

class $modify(PerfMenuLayer, MenuLayer) {
	bool init() {
		if (!MenuLayer::init()) return false;

		static bool created = false;
		if (!created) {
			created = true;
			auto overlay = PerfOverlay::create();
			if (overlay) SceneManager::get()->keepAcrossScenes(overlay);
			applyFps();
		}
		return true;
	}
};

class $modify(PerfGameLayer, GJBaseGameLayer) {
	// Reduce las partículas del nivel según el perfil
	CCParticleSystemQuad* createParticle(int p0, char const* p1, int p2, tCCPositionType p3) {
		auto ret = GJBaseGameLayer::createParticle(p0, p1, p2, p3);
		if (!ret) return ret;

		float f = getCfg().particles;
		if (f >= 1.f) return ret;

		if (f <= 0.f) {
			ret->stopSystem();
			ret->setVisible(false);
		} else {
			ret->setEmissionRate(ret->getEmissionRate() * f);
		}
		return ret;
	}

	// Quita el shake de cámara si el perfil lo pide
	void shakeCamera(float p0, float p1, float p2) {
		if (getCfg().noShake) return;
		GJBaseGameLayer::shakeCamera(p0, p1, p2);
	}
};
